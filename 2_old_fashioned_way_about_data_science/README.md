# Old-fashioned Way About Data Science

- [Publish Date](#publish-date)
- [Update History](#update-history)
- [Stat over RFCs](#all-you-need-is-text)
- [Drawing in terminal](#drawing-in-terminal)
- [References](#references)

The usual Pythonic way to get a data series is sometimes not the fastest. In some cases, it's enough to have only 3-6 Linux CLI tools to get a necessary data from Web and parse it in a proper way. Let's look at such an approach on how to count the number of released RFC's per year and draw the graph in a terminal.

## Publish Date
* 2019-11-03

## Update History
* 2022-02-06: Improved escaping of special characters in awk delimiter as this didn't work for the gawk implementation
* 2023-06-23: Update charts to the latest available data
* 2024-08-10: Previously used data source is not available anymore, the implementation of data preparation has changed, update charts to the latest available data

### Stat over RFCs
The important question at the start is always "Where can I get the data source?". Fortunately, in 2024<sup>th</sup> you can find the organized list for [all of RFCs](https://www.rfc-editor.org/) <sup id="a1">[1](#f1)</sup>.

<details>
  <summary>Outdated implementation, data source is not available anymore</summary>

At the first glance it seems that we need to parse HTML or XML data, but it's not. Actually, we need to parse a simple text/plain data which should be in the same format for all entries. Here comes our first helper - [w3m](http://w3m.sourceforge.net/) <sup id="a2">[2](#f2)</sup> CLI text-based browser.
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | head -50
RFC Index

...
avoided for brevity
...

See the RFC Editor Web page for more information.

RFC Index

Num  Information
0001 Host Software S. Crocker [ April 1969 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0001)
0002 Host software B. Duvall [ April 1969 ] (TXT, PDF, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0002)
0003 Documentation conventions S.D. Crocker [ April 1969 ] (TXT, HTML) (Obsoleted-By RFC0010) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0003)
```
That's how we find strictly-formed entries as follows:

> `NUMBER DESCRIPTION [ DATE ] OTHER_TECHNICAL_INFO`

and we can extract the date information using [awk](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) <sup id="a3">[3](#f3)</sup> - a DSL designed for text processing. I prepared a test file from a small fragment of the collected data:
```bash
~> cat test-entries
0506 FTP command naming problem M.A. Padlipsky [ June 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0506)
0507 Not Issued
0508 Real-time data transmission on the ARPANET L. Pfeifer, J. McAfee [ May 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0508)
0509 Traffic statistics (April 1973) A.M. McKenzie [ April 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0509)

~> awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries
 June 1973 
 May 1973 
 April 1973 
```
Please note that `^[0-9][0-9][0-9][0-9]` is used instead of `^[0-9]{4}` because [mawk](https://invisible-island.net/mawk/) <sup id="a4">[4](#f4)</sup> [doesn't support](https://github.com/ThomasDickey/original-mawk/issues/25) <sup id="a5">[5](#f5)</sup> repetition.

Since now we able to extract human-readable dates, let's convert them to numeric format with [date](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html) <sup id="a6">[6](#f6)</sup> utility during implicit loop given from [xargs](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/xargs.html) <sup id="a7">[7](#f7)</sup>:
```bash
~> # Years extraction
~> awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%Y"
1973
1973
1973
~> # Months extraction
~> awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m"
06
05
04
```
The next simple step is sorting ([sort](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/sort.html) <sup id="a8">[8](#f8)</sup> tool) and counting unique ([uniq](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/uniq.html) <sup id="a9">[9](#f9)</sup> tool) values:
```bash
~> awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%Y" | sort -n | uniq -c
      3 1973

~> awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m" | sort -n | uniq -c
      1 04
      1 05
      1 06
```
We're ready to put it all together over pipes:
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\\[|\\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%Y" | sort -n | uniq -c
      1 1968
     25 1969
     58 1970
    182 1971
    134 1972
    162 1973
     60 1974
     24 1975
     11 1976
     20 1977
      8 1978
      7 1979
     17 1980
     29 1981
     37 1982
     49 1983
     39 1984
     41 1985
...
```
</details>
<br/>

Over the years, the old HTML-to-TXT-based easily parsable RFC index format has gone, and the only applicable way to gather all existing issued RFCs has become an [XML-based index](https://www.ietf.org/rfc/rfc-index.xml) <sup id="a14">[14](#f14)</sup>. In some ways this is even a bit easier, since XML is known as a structured and stable M2M format (and old enough).

Classic helper to parse XML in CLI is [xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) <sup id="a15">[15](#f15)</sup>. Trying to collect the XML-based index using [curl](https://curl.se/) <sup id="a16">[16](#f16)</sup> and showing first 5 lines of the document:

```bash
~> curl -s https://www.rfc-editor.org/rfc-index.xml | head -5
<?xml version="1.0" encoding="UTF-8"?>
<rfc-index xmlns="https://www.rfc-editor.org/rfc-index"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://www.rfc-editor.org/rfc-index
                               https://www.rfc-editor.org/rfc-index.xsd">
```

First of all, we need to analyze the XML format and where the required data is located. After the short investigation it's clear, the path is the following:
```
rfc-index
└── rfc-entry
    └── date
        ├── day
        ├── month
        └── year
```
Where the day is an optional field.

It's also important to note the custom XML namespace defined in the document's 2nd line, it will prevent xmllint from [XPath-based](https://www.tutorialspoint.com/evaluate-xpath-in-the-linux-command-line) <sup id="a17">[17](#f17)</sup> parsing. To avoid overengineering by fighting the XML namespace, it is easier to remove it altogether:

```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | head -5
<?xml version="1.0" encoding="UTF-8"?>
<rfc-index >
  <bcp-entry>
    <doc-id>BCP0001</doc-id>
  </bcp-entry>
```
Looks much better, and now is the time to try to extract all the years::

```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/year/text()" - | sort -n | uniq -c
      1 1968
     25 1969
     58 1970
    182 1971
    134 1972
    162 1973
     60 1974
     24 1975
     11 1976
     20 1977
      8 1978
      7 1979
     17 1980
     29 1981
     37 1982
     49 1983
     39 1984
     41 1985
...
```

## Drawing in terminal
Bare numbers aren't very comfortable for analysis and thus we'll use [gnuplot](http://www.gnuplot.info/) <sup id="a10">[10](#f10)</sup> utility to draw graphs in the following [configuration](http://www.bersch.net/gnuplot-doc/gnuplot.html) <sup id="a11">[11](#f11)</sup>:

```bash
gnuplot -e "set term dumb size 170, 35; set xtics 3; plot '-' with lines notitle"
```
It'll read the STDIN stream and draw a 170x35 graph right in the terminal. Putting it all together one more time:

```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/year/text()" - | sort -n | uniq -c | gnuplot -e "set term dumb size 170, 35; set xtics 3; plot '-' with lines notitle"


  2030 +--------------------------------------------------------------------------------------------------------------------------------------------------------------+
       |++++++++++++ ++++++++++++++++++++++++++ +++++++++++++++++++++++++ ++++++++++++++++++++++++++ +++++++++++++++++++++++++ ++++++++++++++++++++++++++ ++++++++++++|
       |                                                                                                                                                              |
       |                                *******************************                                                                                               |
  2020 |-+                                                             ********************                                                                         +-|
       |                                                             ********************                                                                             |
       |                                                                                 **************************                                                   |
       |                                                                                                       **********                                             |
       |                                                                                              *****************************************                       |
  2010 |-+                                                                                                ********************************                          +-|
       |                                                                                                   ************************************                       |
       |                                                                                                        ******************************************************|
       |                                                                       *********************************                                                      |
  2000 |-+                                                                *******************************                                                           +-|
       |                                                                         ********************                                                                 |
       |                                            *****************************                                                                                     |
       |                                              ******************                                                                                              |
       |                          ********************                                                                                                                |
  1990 |-+              **********                                                                                                                                  +-|
       |       *********                                                                                                                                              |
       |           ****                                                                                                                                               |
       |           ******                                                                                                                                             |
  1980 |-+ ********                                                                                                                                                 +-|
       | ******                                                                                                                                                       |
       |   ***********                                                                                                                                                |
       |              ******************************************                                                                                                      |
       |                                         **********************                                                                                               |
  1970 |*****************************************                                                                                                                   +-|
       |                                                                                                                                                              |
       |                                                                                                                                                              |
       |++++++++++++ ++++++++++++++++++++++++++ +++++++++++++++++++++++++ ++++++++++++++++++++++++++ +++++++++++++++++++++++++ ++++++++++++++++++++++++++ ++++++++++++|
  1960 +--------------------------------------------------------------------------------------------------------------------------------------------------------------+
                   3                         1                         1                          2                         3                          4             459
```
As you can see, something is going wrong. This is because gnuplot expects the first column as the X-axis and the second as the Y-axis. We need to swap our columns with each other:
```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/year/text()" - | sort -n | uniq -c | awk '{print $2" "$1}'
1968 1
1969 25
1970 58
1971 182
1972 134
1973 162
1974 60
1975 24
1976 11
1977 20
1978 8
1979 7
1980 17
1981 29
1982 37
1983 49
1984 39
1985 41
...
```
And rerun our graph plotting:
```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/year/text()" - | sort -n | uniq -c | awk '{print $2" "$1}' | gnuplot -e "set term dumb size 170, 35; set xtics 3; plot '-' with lines notitle"


  500 +---------------------------------------------------------------------------------------------------------------------------------------------------------------+
      |       +        +       +        +       +        +       +       +        +       +        +       +       +        +       +        +       +        +       |
      |                                                                                                                                                               |
  450 |-+                                                                                                        *                                                  +-|
      |                                                                                                          *                                                    |
      |                                                                                                         * *                                                   |
  400 |-+                                                                                                       * *                                                 +-|
      |                                                                                                         * *           **                                      |
      |                                                                                                        *  *         **  *                                     |
  350 |-+                                                                                                      *   *       *     *                                  +-|
      |                                                                                                       *    *       *      *                                   |
      |                                                                                                       *    *      *        *   **                             |
  300 |-+                                                                                                    *      **    *        *  *  *****                      +-|
      |                                                                                                     *         ****          **        *                       |
      |                                                                                        **          *                        *          *                      |
  250 |-+                                                                                   ***  *       **                                     *                   +-|
      |                                                                                   **     *     **                                        *         *          |
      |                                                                                  *        *  **                                          *       ** *         |
      |                                                                                 *         * *                                             *     *    *        |
  200 |-+                                                                             **           *                                               ** **      *     +-|
      |       *                                                             ****     *                                                               *         **     |
      |       **    *                                                      *    *   *                                                                            *    |
  150 |-+    *  * ** *                                                     *     * *                                                                             *  +-|
      |      *   *   *                                                    *       *                                                                               *   |
      |      *        *                                                   *                                                                                       *   |
  100 |-+    *        *                                                ***                                                                                         *+-|
      |     *          *                                             **                                                                                               |
      |     *          *                                           **                                                                                                 |
   50 |-+ **            *                      ***   **    ********                                                                                                 +-|
      |  *               *               ******   ***  ** *                                                                                                           |
      |**     +        +  *******      **       +        *       +       +        +       +        +       +       +        +       +        +       +        +       |
    0 +---------------------------------------------------------------------------------------------------------------------------------------------------------------+
     1968    1971     1974    1977     1980    1983     1986    1989    1992     1995    1998     2001    2004    2007     2010    2013     2016    2019     2022    2025
```
Now it looks correct.

To draw the statistics over months, some additional data augmentation is required. To convert month names:
```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/month/text()" - | sort -n | uniq -c | awk '{print $2" "$1}'
April 848
August 860
December 630
February 802
January 828
July 696
June 847
March 878
May 796
November 659
October 820
September 751
```

to numeric format with [date](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html) <sup id="a6">[6](#f6)</sup> utility during implicit loop given from [xargs](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/xargs.html) <sup id="a7">[7](#f7)</sup>:
```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/month/text()" - | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m" | sort
 -n | uniq -c | awk '{print $2" "$1}'
01 828
02 802
03 878
04 848
05 796
06 847
07 696
08 860
09 751
10 820
11 659
12 630
```

For months statistics we see the expected deviations for July and November/December, the most productivity release dates are in March/April:
```bash
~> curl -s https://www.ietf.org/rfc/rfc-index.xml | xmllint --format - | sed '2 s/xmlns=".*"//g' | xmllint --xpath "//rfc-entry/date/month/text()" - | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m" | sort
 -n | uniq -c | awk '{print $2" "$1}' | gnuplot -e "set term dumb size 170, 35; set xtics 1; plot '-' with lines notitle"


  900 +---------------------------------------------------------------------------------------------------------------------------------------------------------------+
      |              +             +              +             +              +             +              +             +              +             +              |
      |                            ***                                                                                                                                |
      |                          **   *****                                                                                                                           |
      |                        **          *****                                                            *                                                         |
  850 |-+                    **                 ****                          **                           * *                                                      +-|
      |                     *                       **                      **  *                         *   **                                                      |
      |**                 **                          **                 ***     *                       *      *                                                     |
      |  *****          **                              ***            **         *                     *        *                      **                            |
      |       *****   **                                   **       ***            *                    *         *                   **  *                           |
  800 |-+          ***                                       **   **               *                   *           **               **     *                        +-|
      |                                                        ***                  *                 *              *            **       *                          |
      |                                                                              *               *                *         **          *                         |
      |                                                                               *             *                  **     **             *                        |
      |                                                                                *           *                     *  **                *                       |
  750 |-+                                                                               *         *                       **                   *                    +-|
      |                                                                                  *       *                                              *                     |
      |                                                                                   *      *                                              *                     |
      |                                                                                   *     *                                                *                    |
      |                                                                                    *   *                                                  *                   |
      |                                                                                     * *                                                    *                  |
  700 |-+                                                                                    *                                                      *               +-|
      |                                                                                                                                              *                |
      |                                                                                                                                              *                |
      |                                                                                                                                               *               |
      |                                                                                                                                                ***            |
  650 |-+                                                                                                                                                 *****     +-|
      |                                                                                                                                                        *****  |
      |                                                                                                                                                             **|
      |                                                                                                                                                               |
      |              +             +              +             +              +             +              +             +              +             +              |
  600 +---------------------------------------------------------------------------------------------------------------------------------------------------------------+
      1              2             3              4             5              6             7              8             9              10            11             12
```

Moreover, we can plot the same graph for [IETF](https://ietf.org/) <sup id="a12">[12](#f12)</sup> [Internet-Drafts](https://www.ietf.org/standards/ids/) <sup id="a13">[13](#f13)</sup> to discover how rapidly their numbers are growing:
```bash
~> curl -s https://mirror.funkfreundelandshut.de/ietf/internet-drafts/all_id.txt | awk '/^draft/{print $2}' | awk -F- '!/RFC/{print $1}' | sort -n | uniq -c | awk '{print $2" "$1}' | gnuplot -e "set term dumb size 170, 35; set xtics 1; set ytics 20; plot '-' notitle smooth csplines"


  2400 +--------------------------------------------------------------------------------------------------------------------------------------------------------------+
  2360 |-+  +   +    +   +    +   +    +   +    +   +    +    +   +    +   +    +   +    +   +    +   +    +   +    +    +   +    +   +    +   +    +   +    +   +  +-|
  2280 |-+                                                                                                                                                          +*|
  2200 |-+                                                                                                                                                          +*|
  2120 |-+                                                                                                                                                          +*|
  2040 |-+                                                                                                                                                          *-|
  1960 |-+                                                                                                                                                          *-|
  1880 |-+                                                                                                                                                          *-|
  1800 |-+                                                                                                             ***                                          *-|
  1740 |-+                                                                                                          ***   ***                                      *+-|
  1660 |-+                                                                                              ****    ****         ***                                   *+-|
  1580 |-+                                                    *******                                ***    ****                **                                 *+-|
  1500 |-+                                                   *       **                             *                             **                               *+-|
  1420 |-+                                                  *          ****    ********           **                                *                             * +-|
  1340 |-+                                                 *               ****        ***     ***                                   **                           * +-|
  1260 |-+                                                *                               *****                                        ********                  *  +-|
  1180 |-+                                              **                                                                                     **                *  +-|
  1120 |-+                                             *                                                                                         *           ****   +-|
  1040 |-+                                         ****                                                                                           ***       *       +-|
   960 |-+                                        *                                                                                                  *******        +-|
   880 |-+                                      **                                                                                                                  +-|
   800 |-+                                     *                                                                                                                    +-|
   720 |-+                                 ****                                                                                                                     +-|
   640 |-+                                *                                                                                                                         +-|
   580 |-+                              **                                                                                                                          +-|
   500 |-+                           ***                                                                                                                            +-|
   420 |-+                         **                                                                                                                               +-|
   340 |-+                   ******                                                                                                                                 +-|
   260 |-+              *****                                                                                                                                       +-|
   180 |-+         *****                                                                                                                                            +-|
   100 |-+ ********  +   +    +   +    +   +    +   +    +    +   +    +   +    +   +    +   +    +   +    +   +    +    +   +    +   +    +   +    +   +    +   +  +-|
    20 +--------------------------------------------------------------------------------------------------------------------------------------------------------------+
      1989 199 1991 199 1993 199 1995 199 1997 199 1999 2000 200 2002 200 2004 200 2006 200 2008 200 2010 201 2012 2013 201 2015 201 2017 201 2019 202 2021 202 2023 2024
```


## References
<b id="f1">1</b>. [RFC Editor](https://www.rfc-editor.org/) [↩](#a1)<br/>
<b id="f2">2</b>. [Text-based web browser w3m](http://w3m.sourceforge.net/) [↩](#a2)<br/>
<b id="f3">3</b>. ["Aho, Weinberger and Kernighan" domain-specific language](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) [↩](#a3)<br/>
<b id="f4">4</b>. [awk originally written by Mike Brennan](https://invisible-island.net/mawk/) [↩](#a4)<br/>
<b id="f5">5</b>. [Built-in regex's do not support brace-expressions](https://github.com/ThomasDickey/original-mawk/issues/25) [↩](#a5)<br/>
<b id="f6">6</b>. [date - write the date and time](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html) [↩](#a6)<br/>
<b id="f7">7</b>. [xargs - construct argument lists and invoke utility](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/xargs.html) [↩](#a7)<br/>
<b id="f8">8</b>. [sort - sort, merge, or sequence check text files](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/sort.html) [↩](#a8)<br/>
<b id="f9">9</b>. [uniq - report or filter out repeated lines](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/uniq.html) [↩](#a9)<br/>
<b id="f10">10</b>. [gnuplot - portable command-line driven graphing utility](http://www.gnuplot.info/) [↩](#a10)<br/>
<b id="f11">11</b>. [gnuplot documentation](http://www.bersch.net/gnuplot-doc/gnuplot.html) [↩](#a11)<br/>
<b id="f12">12</b>. [Internet Engineering Task Force](https://ietf.org/) [↩](#a12)<br/>
<b id="f13">13</b>. [Internet-Drafts](https://www.ietf.org/standards/ids/) [↩](#a13)<br/>
<b id="f14">14</b>. [XML-based RFCs index](https://www.ietf.org/rfc/rfc-index.xml) [↩](#a14)<br/>
<b id="f15">15</b>. [xmllint - command line XML tool](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) [↩](#a15)<br/>
<b id="f16">16</b>. [curl - command line tool and library for transferring data with URL syntax](https://curl.se/) [↩](#a16)<br/>
<b id="f17">17</b>. [Evaluate XPath in the Linux Command Line](https://www.tutorialspoint.com/evaluate-xpath-in-the-linux-command-line) [↩](#a17)<br/>
