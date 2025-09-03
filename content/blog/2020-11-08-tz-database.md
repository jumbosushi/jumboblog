+++
title = "How the tz database works"
date = "2020-11-08"
slug = "tz-database"
+++

The other day I ran into [this timezone issue](https://github.com/tzinfo/tzinfo/issues/120) in Ruby, and it exposed me to the concept of the tz database that I didn't know before. There were almost no blog posts explaining how it works, so I decided to write one myself.

We'll be using the `alpine:3.12` Docker image to test things out.

## What is it?

The [tz database](https://en.wikipedia.org/wiki/Tz_database) is a standardized collection of timezone data and rules used by most systems worldwide to handle timezone conversions.

Here are the [official documentation](https://data.iana.org/time-zones/tz-link.html) and [the source code on GitHub](https://github.com/eggert/tz). The [`NEWS`](https://github.com/eggert/tz/blob/master/NEWS) file is used as a replacement for the more common `CHANGELOG` in other projects.

## tzdata package

The tz database and its related CLI tools are distributed as the `tzdata` package. Let's download it within our container:

```txt
$ docker run -it alpine:3.12
/ # apk add tzdata
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/community/x86_64/APKINDEX.tar.gz
(1/1) Installing tzdata (2020c-r0)
Executing busybox-1.31.1-r19.trigger
OK: 9 MiB in 15 packages
```

To check which version your `tzdata` package is on, use the package manager's `version` command:

```txt
/ # apk version tzdata
Installed:           Available:
tzdata-2020c-r0      = 2020c-r0
```

By default, it saves timezone files to the `/usr/share/zoneinfo` directory:

```txt
/ # ls /usr/share/zoneinfo
Africa/  Asia/  CET  Cuba  Egypt ...
```

## CLI tools

When you install the `tzdata` package, several related CLI tools are included:

### zic (Zone Information Compiler)

`zic` compiles source text files into [Time Zone Information Format (TZif)](https://man7.org/linux/man-pages/man5/tzfile.5.html) files. We'll look at how to use this later in the article.

### zdump

`zdump` is used to read TZif files:

```txt
/ # zdump /usr/share/zoneinfo/America/Los_Angeles
/usr/share/zoneinfo/America/Los_Angeles  Sun Nov  8 17:05:59 2020 PST
```

`/etc/localtime` is used by `libc` and many other systems to determine the system's timezone. Under the hood, it should be a symbolic link to one of the TZif files in `/usr/share/zoneinfo` as documented [here](https://www.freedesktop.org/software/systemd/man/localtime.html). `zdump` can also be used to examine this file (it defaults to GMT in Alpine):

```txt
/ # zdump /etc/localtime
/etc/localtime  Mon Nov  9 01:25:48 2020 GMT
```

## Timezone source files

Although timezone rules have become relatively stable within the last decade, this wasn't the case in the 20th century. Here's an example:

```txt
/ # TZ=America/Los_Angeles date --date="1950-04-01"
Sat Apr  1 00:00:00 PST 1950
/ # TZ=America/Los_Angeles date --date="2007-04-01"
Sun Apr  1 00:00:00 PDT 2007
```

As captured in [the comment](https://github.com/eggert/tz/blob/beba17f43925823308c6f7f0d5ca9b52d00d351f/northamerica#L525), California voted to start Daylight Saving Time from the last Sunday in April starting in 1950. However, the federal government [passed a bill](https://en.wikipedia.org/wiki/Daylight_saving_time_in_the_United_States#2005%E2%80%932009:_Second_extension) to extend Daylight Saving Time to start from the second Sunday of March in 2007. Therefore, you can see that April 1st was not in Daylight Saving Time back in 1950.

Policy changes like this happened throughout the world, and the tz database documents these events in its source files. The tz database source files are composed of three main components: rules, zones, and links. Here is an example configuration from the [`zic` man page](https://man7.org/linux/man-pages/man8/zic.8.html) (no need to understand what each line means! This is meant to give an idea of what it may look like):

```txt
# Rule  NAME  FROM  TO    TYPE  IN   ON       AT    SAVE  LETTER/S
Rule    Swiss 1941  1942  -     May  Mon>=1   1:00  1:00  S
Rule    Swiss 1941  1942  -     Oct  Mon>=1   2:00  0     -
Rule    EU    1977  1980  -     Apr  Sun>=1   1:00u 1:00  S
Rule    EU    1977  only  -     Sep  lastSun  1:00u 0     -
Rule    EU    1978  only  -     Oct   1       1:00u 0     -
Rule    EU    1979  1995  -     Sep  lastSun  1:00u 0     -
Rule    EU    1981  max   -     Mar  lastSun  1:00u 1:00  S
Rule    EU    1996  max   -     Oct  lastSun  1:00u 0     -

# Zone  NAME           STDOFF      RULES  FORMAT  [UNTIL]
Zone    Europe/Zurich  0:34:08     -      LMT     1853 Jul 16
                      0:29:45.50  -      BMT     1894 Jun
                      1:00        Swiss  CE%sT   1981
                      1:00        EU     CE%sT

# Link  TARGET         LINK-NAME
Link    Europe/Zurich  Europe/Vaduz
```

A simplified overview of these lines:
- **Rule** defines when and how to adjust time (like "start daylight saving on the first Sunday in March")
- **Zone** defines a geographic timezone by specifying its base UTC offset and which rules apply to it
- **Link** creates an alias so multiple names can refer to the same timezone (Europe/Vaduz will use the same rules as Europe/Zurich)

More examples are available on the [tz-how-to page](https://data.iana.org/time-zones/tz-how-to.html).

## Creating our custom timezone

After getting a grasp of the basic syntax, we can define a new timezone ourselves! We'll use the fictional [Konoha city in Hi No Kuni](https://naruto.fandom.com/wiki/Land_of_Fire) from Naruto as our example timezone name. Run the following command to create a source file in the container:

```sh
echo "
# Hi No Kuni
# Rule  NAME    FROM    TO      -       IN      ON      AT      SAVE    LETTER/S
Rule    HI      2020    only    -       Mar     Sun>=8  0:00    1:00    D
Rule    HI      2020    only    -       Nov     1       0:00    0       S

# Zone  NAME              STDOFF  RULES   FORMAT  [UNTIL]
Zone    Hi_No_Kuni/Konoha -8:00   HI      K%sT
" > /konoha.zi
```

The above config defines the following:
- The `Hi_No_Kuni/Konoha` zone will be 8 hours behind UTC (same as PST). It will use the `HI` rules. The `%s` substring in `K%sT` will be replaced with the `LETTER` field from the current rule in use.
- We define two rules:
  1. From the 2nd Sunday in March 2020, subtract 1 hour from the local time. Use `KDT` to refer to this timezone.
  2. From November 1st, 2020, use the exact local time. Use `KST` to refer to this timezone.

Let's verify that it works:

```txt
/ # zic /konoha.zi
/ # zdump /usr/share/zoneinfo/Hi_No_Kuni/Konoha
/usr/share/zoneinfo/Hi_No_Kuni/Konoha  Sun Nov  8 19:07:59 2020 KST
/ # ln -sf /usr/share/zoneinfo/Hi_No_Kuni/Konoha /etc/localtime
/ # date --date="2020-03-14"
Sat Mar 14 00:00:00 KDT 2020
/ # date --date="2020-11-01"
Sun Nov  1 00:00:00 KST 2020
```

The `zic` compiler saves the output to the `/usr/share/zoneinfo` path based on the zone name. At the time of writing this post, it is 11/08/2020. As expected, `zdump` prints the time in `KST` (possibly known as `Konoha Standard Time` in Naruto's world). We're setting `Hi_No_Kuni/Konoha` as the default timezone in this container by creating a symbolic link to `/etc/localtime`. As a result, the `date` command prints the correct timezone for any given date!

## Conclusion

I learned a lot while writing this post, but there is so much more to the tz database. As praised in [this blog post](https://blog.jonudell.net/2009/10/23/a-literary-appreciation-of-the-olsonzoneinfotz-database/), the documentation in the source code is a wealth of knowledge about everything timezone-related. I highly recommend checking out what kind of rules are applied in your home country.

Hope this helped you understand the tz database a little better!
