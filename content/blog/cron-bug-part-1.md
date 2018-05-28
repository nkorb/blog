---
Title: Hunting a bug in Debian's cron package (Part I)
Date: 2018-05-19
---

*I started writing this post initially about a bug (is it really a bug?) in
Debian's cron implementation (Vixie Cron plus some patches). Along the way, I
noticed that a fix for this "bug" might not be easy...*

One annoying thing when getting emails from cron, is that you somethimes you
cannot determine which is the problematic host. At work we use a naming scheme
for some hosts which looks like this: `$role$number.$env.$service.grnet.gr`.
As you notice, *naming things is really hard.*

So, let's assume that we have two nodes which are hosting a Varnish cache for
service foo. We would name these hosts like this:

* varnish0.prod.foo.grnet.gr
* varnish0.stg.foo.grnet.gr

Hosts have a bad reputation of sending email notifications when a cronjob
fails:

```
Cron <root@varnish0> test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
```

But there's a problem: without opening up all headers, just with a first look,
you cannot tell which host actually sent that email. This is maybe not a
problem if you own a Unix beard, *but it can be a problem* for helpdesk people.

Anyway, we noticed that Debian has added a command line option to its cron
implementation which includes the hostname, plus the domain name in the subject
when sending emails; something that most people would call a FQDN.

This new feature can be enabled using the `-n` flag. So our first thought was
to enable the option all over our infrastructure and let's wait for the first
cronjob to crash. To our suprise, the email's subject was:

```
Cron <root@varnish0> test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
```
Damn. Something went wrong. Maybe puppet was disabled on this host. Maybe we
did a typo on our puppet manifest. Nope, everything seemed fine. Also, let me
add at this point, that all our managed have their FQDN declared in `/etc/hosts`:

```
$ grep varnish0.stg.foo.grnet.gr /etc/hosts
1.2.3.4 varnish0.stg.foo.grnet.gr varnish0
1:2:3::4 varnish0.stg.foo.grnet.gr varnish0

$ cat /etc/hostname
varnish0

$ hostname 
varnish0

$ hostname -f
varnish0.stg.foo.grnet.gr
```

After checking that everything was OK, I decided to dig a little bit further
into cron's source code to see what's happening under the hood. This *must*
be some kind of a bug.

A small sidenote here: Debian's `cron(8)` is based on Vixie Cron, which does not
include a `-n` flag. So this flag must be a Debian-specific change

Let's get our hands dirty:

```
$ apt-get source cron
$ vim cron-3.0pl1/cron.c
[snip]
static void
parse_args(argc, argv)
  int argc;
  char  *argv[];
{
  int argch;

  log_level = 1;
  stay_foreground = 0;
        lsbsysinit_mode = 0;
        fqdn_in_subject = 0;

  while (EOF != (argch = getopt(argc, argv, "lfnx:L:"))) {
    switch (argch) {
    default:
      usage();
    case 'f':
      stay_foreground = 1;
      break;
    case 'x':
      if (!set_debug_flags(optarg))
        usage();
      break;
    case 'l':
      lsbsysinit_mode = 1;
      break;
    case 'n':
      fqdn_in_subject = 1;
      break;
    case 'L':
      log_level = atoi(optarg);
      break;
    }   
  }
}
[/snip]
```

I'm looking for the guy who uses the `fqdn_in_subject` variable. After some
grepping, I found a mention of the variable at `do_command.c`:

```
563:       fprintf(mail, "Subject: Cron <%s@%s> %s%s\n",
564:                       usernm,
565:                       fqdn_in_subject ? hostname : first_word(hostname, "."),
566:                       e->cmd, status?" (failed)":"");
```

In our case `fqdn_in_subject == 1`, so it will print the content of variable
`hostname`. Let's find out how this variable gets populated. Again,
at `do_command.c`:

```
554:       (void) gethostname(hostname, MAXHOSTNAMELEN);
```

Hm. I can see that `MAXHOSTNAMELEN == 64`. But what does `gethostname(2)`
actually do?

```
$ man 2 gethostname
<snip>
gethostname()  returns  the  null-terminated hostname in the character array
name, which has a length of len bytes.  If the null-terminated hostname is too
large to fit, then the name is truncated, and no error is returned (but see
NOTES below).  POSIX.1-2001 says that if such truncation occurs, then it is
unspecified whether the returned buffer includes a terminating null byte.
</snip>
```

I wanted to observe the behavior of this function in my system. I wrote this
small C program in order to do that:

```
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    char h[64];
    gethostname(h, 64);
    printf("%s\n", h);
    return 0;
}
```

Let's run it:
```
varnish0.stg.foo.grnet.gr # ./hostname_test
varnish0
varnish0.stg.foo.grnet.gr # ltrace ./hostname_test
__libc_start_main(0x400556, 1, 0x7fffe4eb7458, 0x400590 <unfinished ...>
gethostname("varnish0", 64) = 0
puts("varnish0"varnish0) = 9
+++ exited (status 0) +++
```

As we can see, `gethostname(2)` returns the hostname of the node and not the
FQDN, as expected. That's a good sign, implying that there's not a
misconfiguration on our fleet.

But, after taking another look at `do_command.c`, I noticed this:
```
565:                       fqdn_in_subject ? hostname : first_word(hostname, "."),
```

But what does this function do?

```
$ cat cron-3.0pl1/misc.c
#define MAX_TEMPSTR 1000

char *
first_word(s, t)
  register char *s; /* string we want the first word of */
  register char *t; /* terminators, implicitly including \0 */
{
  static char retbuf[2][MAX_TEMPSTR + 1]; /* sure wish C had GC */
  static int retsel = 0;
  register char *rb, *rp;

  /* select a return buffer */
  retsel = 1-retsel;
  rb = &retbuf[retsel][0];
  rp = rb; 

  /* skip any leading terminators */
  while (*s && (NULL != strchr(t, *s))) {
    s++;
  }

  /* copy until next terminator or full buffer */
  while (*s && (NULL == strchr(t, *s)) && (rp < &rb[MAX_TEMPSTR])) {
    *rp++ = *s++;
  }

  /* finish the return-string and return it */
  *rp = '\0';
  return rb; 
}
```

My knowledge of C might not be the best one, but I understand that this
function gets two char pointers as arguments and returns the first substring
before the first occurence of `char *t`.

Considering all of the above, I suspect that in the following expression
expression
```
565:                       fqdn_in_subject ? hostname : first_word(hostname, "."),
```
the developer wanted this expression to evaluate to a string of the FQDN
if `fqdn_in_subject` is true. Otherwise, it should return anything that is
before the dot (the hostname). It's obvious that this piece of code contains
a bug that needs to be fixed.

After taking a quick look at the package's git tree, it seems that the original
developer of cron assumed that `gethostname(2)` would return the FQDN of the
current node. Even before the addition of the `-n` flag, cron used
the `first_word()` function to strip of the domain of the node.

```
commit faaeed16d9639112f2a18072300619281abb2e95
Author: Christian Kastner <debian@kvr.at>
Date:   Thu Oct 9 23:31:18 2014 +0200

Add an option -n to include FQDN in mail subject

Closes: #570423

<snip>
diff --git a/do_command.c b/do_command.c
index c22d9d6..d52ec33 100644
--- a/do_command.c
+++ b/do_command.c
@@ -561,7 +561,8 @@ child_process(e, u)
fprintf(mail, "From: root (Cron Daemon)\n");
fprintf(mail, "To: %s\n", mailto);
fprintf(mail, "Subject: Cron <%s@%s> %s%s\n",
-                       usernm, first_word(hostname, "."),
+                       usernm,
+                       fqdn_in_subject ? hostname : first_word(hostname, "."),
</snip>
```

After reaching this point, I understood that it was not easy to resolve this
bug. Why did both developers assume that `gethostname(2)` returns an FQDN? Is
this a unnoticed bug or am I missing something? Which is the correct way to
find the host's current FQDN? What happens if the FQDN is not defined in
`/etc/hosts`? What happens with NIS domain names? Should we care or not?

*To be continued...*
