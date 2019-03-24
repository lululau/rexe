---
title: The `rexe` Command Line Executor and Filter
date: 2019-02-15
---

_[Caution: This is a long article! If you lose patience reading it, I suggest skimming the headings
and the source code, and at minimum, reading the Conclusion.]_

----

Shell scripting is great for simple tasks but for anything nontrivial
it can easily get cryptic and awkward (pun intended!).
 
Often, I solve this problem by writing a Ruby script instead. Ruby gives me fine
grained control in a language that is all about clarity, conciseness, and expressiveness.

Unfortunately, when there are multiple OS commands to be called, then Ruby can be
awkward too.

### Using the Ruby Interpreter on the Command Line

Sometimes a good solution is to combine Ruby and shell scripting on the same
command line. Here's an example, using an intermediate environment variable to
simplify the logic and save the data for use by future commands.
An excerpt of the output follows the code:

```
➜  ~   export EUR_RATES_JSON=`curl https://api.exchangeratesapi.io/latest`
➜  ~   echo $EUR_RATES_JSON | ruby -r json -r yaml -e 'puts JSON.parse(STDIN.read).to_yaml'
---
rates:
  MXN: 21.96
  AUD: 1.5964
  HKD: 8.8092
  ...
base: EUR
date: '2019-03-08'
```

However, the configuration setup (the `require`s) along with the reading, parsing, and formatting
make the command long and tedious, discouraging this approach.

### Rexe

The `rexe` script [^1] can simplify such commands.
Among other things, `rexe` provides switch-activated input parsing and output formatting so that converting 
from one format to another is trivial.
The previous `ruby` command can be expressed in `rexe` as:

```
➜  ~   echo $EUR_RATES_JSON | rexe -mb -ij -oy self
```

The command options may seem cryptic, but they're logical so it shouldn't take long to learn them:

* `-mb` - __mode__ to consume all standard input as a single __big__ string
* `-ij` - parse that __input__ with __JSON__; `self` will be the parsed object
* `-oy` - __output__ the final value as __YAML__
 
`rexe` is at https://github.com/keithrbennett/rexe and can be installed with
`gem install rexe`. `rexe` provides several ways to simplify Ruby on the command
line, tipping the scale so that it is practical to do it more often.

----

Here is `rexe`'s help text as of the time of this writing:

```
rexe -- Ruby Command Line Executor/Filter -- v0.13.0 -- https://github.com/keithrbennett/rexe

Executes Ruby code on the command line, optionally automating management of standard input
and standard output, and optionally parsing input and formatting output with YAML, JSON, etc.

rexe [options] 'Ruby source code'

Options:

-c  --clear_options        Clear all previous command line options specified up to now
-f  --input_file           Use this file instead of stdin; autodetects YAML and JSON file extensions
                           If YAML or JSON: parses file in that mode, sets input mode to -mb
-g  --log_format FORMAT    Log format, logs to stderr, defaults to none (see -o for format options)
-h, --help                 Print help and exit
-i, --input_format FORMAT  Input format
                             -ij  JSON
                             -im  Marshal
                             -in  None (default)
                             -iy  YAML
-l, --load RUBY_FILE(S)    Ruby file(s) to load, comma separated;
                             ! to clear all, or precede a name with '-' to remove
-m, --input_mode MODE      Mode with which to handle input (i.e. what `self` will be in your code):
                             -ml  line mode; each line is ingested as a separate string
                             -me  enumerator mode
                             -mb  big string mode; all lines combined into single multiline string
                             -mn  (default) no input mode; no special handling of input; self is an Object.new 
-n, --[no-]noop            Do not execute the code (useful with -g); the following are valid:
                             -n no, -n yes, -n false, -n true, -n n, -n y, -n +, but not -n -
-o, --output_format FORMAT Output format (defaults to puts):
                             -oi  Inspect
                             -oj  JSON
                             -oJ  Pretty JSON
                             -om  Marshal
                             -on  No Output (default)
                             -op  Puts
                             -os  to_s
                             -oy  YAML
-r, --require REQUIRE(S)   Gems and built-in libraries to require, comma separated;
                             ! to clear all, or precede a name with '-' to remove

If there is an .rexerc file in your home directory, it will be run as Ruby code 
before processing the input.

If there is a REXE_OPTIONS environment variable, its content will be prepended to the command line
so that you can specify options implicitly (e.g. `export REXE_OPTIONS="-r awesome_print,yaml"`)
```

### Simplifying the Rexe Invocation

There are two main ways we can make the `rexe` command line even more concise:

* by extracting configuration into the `REXE_OPTIONS` environment variable
* by extracting low level and/or shared code into files that are loaded using `-l`,
  or implicitly with `~/.rexerc`


### The REXE_OPTIONS Environment Variable

The `REXE_OPTIONS` environment variable can contain command line options that would otherwise
be specified on the `rexe` command line:

Instead of this:

```
➜  ~   rexe -r wifi-wand -oa  WifiWand::MacOsModel.new.wifi_info
```

you can do this:

```
➜  ~   export REXE_OPTIONS="-r wifi-wand -oa"
➜  ~   rexe WifiWand::MacOsModel.new.wifi_info
➜  ~   # [more rexe commands with the same options]
```

Putting configuration options in `REXE_OPTIONS` effectively creates custom defaults,
 and is useful when you use options in most or all of your commands. Any options specified on the `rexe`
command line will override the environment variable options.

Like any environment variable, `REXE_OPTIONS` could also be set in your startup script, input on a command line using `export`, or in another script loaded with `source` or `.`.

### Loading Files

The environment variable approach works well for command line _options_, but what if we want to specify Ruby _code_ (e.g. methods) that can be used by your `rexe` code?

For this, `rexe` lets you _load_ Ruby files, using the `-l` option, or implicitly (without your specifying it) in the case of the `~/.rexerc` file. Here is an example of something you might include in such a file:

```
# Open YouTube to Wagner's "Ride of the Valkyries"
def valkyries
  `open "http://www.youtube.com/watch?v=P73Z6291Pt8&t=0m28s"`
end
```

To digress a bit, why would you want this? You might want to be able to go to another room until a long job completes, and be notified when it is done. The `valkyries` method will launch a browser window pointed to Richard Wagner's "Ride of the Valkyries" starting at a lively point in the music. (The `open` command is Mac specific and could be replaced with `start` on Windows, a browser command name, etc.) [^2]
 
 If you like this kind of audio notification, you could download public domain audio files and use a command like player like `afplay` on Mac OS, or `mpg123` or `ogg123` on Linux. This approach is lighter weight, requires no network access, and will not leave an open browser window for you to close.

Here is an example of how you might use the `valkyries` method, assuming the above configuration 
is loaded from your `~/.rexerc` file or an explicitly loaded file:

```
➜  ~   tar czf /tmp/my-whole-user-space.tar.gz ~ ; rexe valkyries
```

(Note that `;` is used rather than `&&` because we want to hear the music whether or not the command succeeds.)

You might be thinking that creating an alias or a minimal shell script for this `open` would be a simpler and more natural
approach, and I would agree with you. However, over time the number of these could become unmanageable, whereas using Ruby
you could build a pretty extensive and well organized library of functionality. Moreover, that functionality could be made available to _all_ your Ruby code (for example, by putting it in a gem), and not just command line one liners.

For example, you could have something like this in a gem or loaded file:

```
def play(piece_code)
  pieces = {
    hallelujah: "https://www.youtube.com/watch?v=IUZEtVbJT5c&t=0m20s",
    valkyries:  "http://www.youtube.com/watch?v=P73Z6291Pt8&t=0m28s",
    wm_tell:    "https://www.youtube.com/watch?v=j3T8-aeOrbg&t=0m1s",
    # ... and many, many more
  }
  `open #{Shellwords.escape(pieces.fetch(piece_code))}`
end
```

...which you could then call like this:

```
➜  ~   tar czf /tmp/my-whole-user-space.tar.gz ~ ; rexe 'play(:hallelujah)'
```

(You need to quote the `play` call because otherwise the shell will process and remove the parentheses.
Alternatively you could escape the parentheses with backslashes.)

One of the examples at the end of this articles shows how you could have different music play for success and failure.


### Logging

A log entry is optionally output to standard error after completion of the code.
The entry is a hash containing the version, date/time of execution, source code
to be evaluated, options (after parsing both the `REXE_OPTIONS` environment variable and the command line),
and the execution time of your Ruby code:
 
```
➜  ~   echo $EUR_RATES_JSON | rexe -gy -ij -oa self
...
---
:count: 0
:rexe_version: 0.13.0
:start_time: '2019-03-21T20:58:51+08:00'
:source_code: ap JSON.parse(STDIN.read)
:options:
  :input_format: :none
  :input_mode: :no_input
  :loads: []
  :output_format: :puts
  :requires:
  - awesome_print
  - json
  - yaml
  :log_format: :yaml
  :noop: false
:duration_secs: 0.070381
``` 

We specified `-gy` for YAML format; there are other formats as well (see the help output or this document)
and the default is `-gn`, which means don't output the log entry at all.

The requires you see were not explicitly specified but were automatically added because
`rexe` will add any requires needed for automatic parsing and formatting, and
we specified those formats in the command line options `-gy -ij -oa`.
 
This extra output is sent to standard error (_stderr_) instead of standard output
(_stdout_) so that it will not pollute the "real" data when stdout is piped to
another command.

If you would like to append this informational output to a file(e.g. `rexe.log`), you could do something like this:

```
➜  ~   rexe ... -gy 2>>rexe.log
```


### Input Modes

`rexe` tries to make it simple and convenient for you to handle standard input, 
and in different ways. Here is the help text relating to input modes:

```
-m, --input_mode MODE      Mode with which to handle input (i.e. what `self` will be in your code):
                           -ml line mode; each line is ingested as a separate string
                           -me enumerator mode
                           -mb big string mode; all lines combined into single multiline string
                           -mn (default) no input mode; no special handling of input; self is not input 
```

The first three are _filter_ modes; they make standard input available
to your code as `self`.

The last (and default) is the _executor_ mode. It merely assists you in
executing the code you provide without any special implicit handling of standard input.

All input modes automatically output to standard output the last value evaluated by your code.
You can suppress automatic output with the `-on` option.


#### -ml "Line" Filter Mode

In this mode, your code would be called once per line of input,
and in each call, `self` would evaluate to each line of text:

```
➜  ~   echo "hello\ngoodbye" | rexe -ms reverse
olleh
eybdoog
```

`reverse` is implicitly called on each line of standard input.  `self`
 is the input line in each call (we could also have used `self.reverse` but the `self.` would have been redundant.).
  
Be aware that, in this mode, although you can control the _content_ of output records, 
there is no way to selectively _exclude_ records from being output. Even if the result of the code
is nil or the empty string, a newline will be output. To prevent this, you can do one of the following:
 
 * use `-me` Enumerator mode and call `select`, `filter`, `reject`, etc.
 * use the `-on` _no output_ mode and call `puts` explicitly for the output you _do_ want


#### -me "Enumerator" Filter Mode

In this mode, your code is called only once, and `self` is an enumerator
dispensing all lines of standard input. To be more precise, it is the enumerator returned by `STDIN.each_line`.

Dealing with input as an enumerator enables you to use the wealth of `Enumerable` methods such as `select`, `to_a`, `map`, etc.

Here is an example of using `-me` to add line numbers to the first 3
files in the directory listing:

```
➜  ~   ls / | rexe -me -on "first(3).each_with_index { |ln,i| puts '%5d  %s' % [i, ln] }"

    0  AndroidStudioProjects
    1  Applications
    2  Desktop
```

Since `self` is an enumerable, we can call `first` and then `each_with_index`.
The `-on` says don't do any automatic output, just the output explicitly specified by `puts` in the source code.


#### -mb "Big String" Filter Mode

In this mode, all standard input is combined into a single (possibly
large) string, with newline characters joining the lines in the string.

A good example of when you would use this is when you need to parse a multiline JSON or YAML representation of an object; 
you need to pass the entire (probably) multiline string to the parse method. 
This is the mode that was used in the first `rexe` example in this article.


#### -mn "No Input" Executor Mode -- The Default

In this mode, no special handling of standard input is done at all;
if you want standard input you need to code it yourself (e.g. with `STDIN.read`).

`self` evaluates to a new instance of `Object`, which would be used 
if you defined methods, constants, instance variables, etc., in your code.


#### Filter Input Mode Memory Considerations

If you may have more input than would fit in memory, you can do the following:

* use `-ml` (line) mode so you are fed only 1 line at a time
* use an Enumerator, either by a) specifying the `-me` (enumerator) mode option,
 or b) using `-mn` (no input) mode in conjunction with something like `STDIN.each_line`. Then: 
  * Make sure not to call any methods (e.g. `map`, `select`)
 that will produce an array of all the input because that will pull all the records into memory, or:
  * use [lazy enumerators](https://www.honeybadger.io/blog/using-lazy-enumerators-to-work-with-large-files-in-ruby/)
 

### Input Formats

`rexe` can parse your input in any of several formats if you like. 
You would request this in the _input format_ (`-i`) option.
Legal values are:

* `-ij` - JSON
* `-im` - Marshal
* `-in` - [None] (default)
* `-iy` - YAML

Except for `-in`, which passes the text to your code untouched, your input will be parsed
in the specified format, and the resulting object passed into your code as `self`.

The input format option is ignored if the input _mode_ is `-mn` ("no input" executor mode, the default),
since there is no preprocessing of standard input in that mode.

### Output Formats

Several output formats are provided for your convenience:

* `-oa` - Awesome Print - calls `.ai` on the object to get the string that `ap` would print
* `-oi` - Inspect - calls `inspect` on the object
* `-oj` - JSON - calls `to_json` on the object
* `-oJ` - Pretty JSON calls `JSON.pretty_generate` with the object
* `-on` - No Output - output is suppressed
* `-op` - Puts - produces what `puts` would output
* `-os` - To String - calls `to_s` on the object
* `-oy` - YAML - calls `to_yaml` on the object

All formats will implicitly `require` anything needed to accomplish their task (e.g. `require 'yaml'`).

You may wonder why these formats are provided, given that their functionality 
could be included in the custom code instead. Here's why:

* The savings in command line length goes a long way to making these commands more readable and feasible.
* It's much simpler to switch formats, as there is no need to change the code itself. This also enables
parameterization of the output format.


### Reading Input from a File

`rexe` also simplifies getting input from a file rather than standard input. The `-f` option takes a filespec
and does with exactly what it would have done with standard input. This shortens:

```
➜  ~   cat filename.ext | rexe ...
```
...to...

```
➜  ~   rexe -f filename.ext ...
```

This becomes even more useful if you are using files whose extensions are `.yml`, `.yaml`, or `.json` (case insensitively).
In this case the input format and mode will be set automatically for you to:

* `-iy` (YAML) or `-ij` (JSON) depending on the file extension
* `-mb` (one big string mode), which assumes that the most common use case will be to parse the entire file at once

So the example we gave above:

```
➜  ~   export EUR_RATES_JSON=`curl https://api.exchangeratesapi.io/latest`
➜  ~   echo $EUR_RATES_JSON | rexe -mb -ij -oy self
```
...could be changed to:

```
➜  ~   curl https://api.exchangeratesapi.io/latest > eur_rates.json
➜  ~   rexe -f eur_rates.json -oy self
``` 

Another possible win for using `-f` is that since it is a command line option, it could be specified in `REXE_OPTIONS`.
This could be useful if you are doing many operations on the same file.

### The $RC Global OpenStruct

For your convenience, the information displayed in verbose mode is available to your code at runtime
by accessing the `$RC` global variable, which contains an OpenStruct. Let's print out its contents using AwesomePrint:
 
```
➜  ~   rexe -oa '$RC'
OpenStruct {
           :count => 0,
    :rexe_version => "0.13.0",
      :start_time => "2019-03-23T20:17:59+08:00",
     :source_code => "$RC",
         :options => {
         :input_format => :none,
           :input_mode => :no_input,
                :loads => [],
        :output_format => :awesome_print,
             :requires => [
            [0] "awesome_print"
        ],
           :log_format => :none,
                 :noop => false
    }
}
``` 
 
Probably most useful in that object
is the record count, accessible with both `$RC.count` and `$RC.i`.
This is only really useful in line mode, because in the others
it will always be 0 or 1. Here is an example of how you might use it as a kind of progress indicator:

```
➜  ~   find / | rexe -ml -on \
'if $RC.i % 1000 == 0; puts %Q{File entry ##{$RC.i} is #{self}}; end'
...
File entry #106000 is /usr/local/Cellar/go/1.11.5/libexec/src/cmd/vendor/github.com/google/pprof/internal/driver/driver_test.go
File entry #107000 is /usr/local/Cellar/go/1.11.5/libexec/src/go/types/testdata/cycles1.src
File entry #108000 is /usr/local/Cellar/go/1.11.5/libexec/src/runtime/os_linux_novdso.go
...
```

Note that a single quote was used for the Ruby code here; 
if a double quote were used, the `$RC` would have been interpreted
and removed by the shell.
  

### Implementing Domain Specific Languages (DSL's)

Defining methods in your loaded files enables you to effectively define a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) for your command line use. You could use different load files for different projects, domains, or contexts, and define aliases or one line scripts to give them meaningful names. For example, if I wrote code to work with Ansible and put it in `~/projects/rexe-ansible.rb`, I could define an alias in my startup script:

```
➜  ~   alias rxans="rexe -l ~/projects/rexe-ansible.rb $*"
```
...and then I would have an Ansible DSL available for me to use by calling `rxans`.

In addition, since you can also call `pry` on the context of any object, you
can provide a DSL in a [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) (shell)
trivially easily. Just to illustrate, here's how you would open a REPL on the File class:

```
➜  ~   ruby -r pry -e File.pry
# or
➜  ~   rexe -r pry File.pry
```

`self` would evaluate to the `File` class, so you could call class methods using only their names:

```
➜  ~   rexe -r pry File.pry

[6] pry(File)> size '/etc/passwd'
6804
[7] pry(File)> directory? '.'
true
[8] pry(File)> file?('/etc/passwd')
true
```

This could be really handy if you call `pry` on a custom object that has methods especially suited to your task:

```
➜  ~   rexe -r  wifi-wand,pry  WifiWand::MacOsModel.new.pry
```

Ruby is supremely well suited for DSL's since it does not require parentheses for method calls, 
so calls to your custom methods _look_ like built in language commands and keywords. 


### Quoting Strings in Your Ruby Code

One complication of using utilities like `rexe` where Ruby code is specified on the shell command line is that
you need to be careful about the shell's special treatment of certain characters. For this reason, it is often
necessary to quote the Ruby code. You can use single or double quotes to have the shell treat your source code
as a single argument. 
An excellent reference for how they differ is [here](https://stackoverflow.com/questions/6697753/difference-between-single-and-double-quotes-in-bash).

Personally, I find single quotes more useful since I usually don't want special characters in my Ruby code  like `$` to be processed by the shell.

When specifying the Ruby code, feel free to fall back on Ruby's super useful `%q{}` and `%Q{}`,
equivalent to single and double quotes, respectively.


### No Op Mode

The `-n` no-op mode will result in the specified source code _not_ being executed. This can sometimes be handy
in conjunction with a `-g` (logging) option; if you have a command ready to go but 
you want to see the configuration options before running it for real.


### Mimicking Method Arguments

You may want to support arguments in your `rexe` commands. 
You could do this by piping in the arguments as `rexe`'s stdin.

One of the previous examples downloaded currency conversion rates. 
To prepare for an example of how to do this, let's find out the available currency codes:

```
➜  /   echo $EUR_RATES_JSON | rexe -ij -mb "self['rates'].keys.sort.join(' ')"
AUD BGN BRL CAD CHF CNY CZK DKK GBP HKD HRK HUF IDR ILS INR ISK JPY KRW MXN MYR NOK NZD PHP PLN RON RUB SEK SGD THB TRY USD ZAR
```

The codes output are the legal arguments that could be sent to `rexe`'s stdin as an argument.
Let's find out the Euro exchange rate for _PHP_, Philippine Pesos:
 
```
➜  ~   echo PHP | rexe -ml -rjson \
        "rate = JSON.parse(ENV['EUR_RATES_JSON'])['rates'][self];\
        %Q{1 EUR = #{rate} #{self}}"

1 EUR = 58.986 PHP
```

In this code, `self` is the currency code `PHP` (Philippine Peso). We have accessed the JSON text to parse from the environment variable we previously populated.

Because we "used up" stdin for the argument, we could not make use of automatic parsing
 of the currency exchange data (using the `-ij` option),
which would have greatly simplified the command. One possible solution to this would be 
to pipe in the JSON or YAML representation of a hash
with entries for both the argument and the currency exchange data...but this might
make the command line too complex to be practical.

### Using the Clipboard for Text Processing

Sometimes when editing text I need to do some one off text manipulation.
Using the system's commands for pasting to and copying from the clipboard,
this can easily be done. On the Mac, the `pbpaste` command outputs to stdout
the clipboard content, and the `pbcopy` command copies its stdin to the clipboard.

Let's say I have the following currency codes displayed on the screen:

```
AUD BGN BRL CAD CHF CNY CZK DKK GBP HKD HRK HUF IDR ILS INR ISK JPY KRW MXN MYR NOK NZD PHP PLN RON RUB SEK SGD THB TRY USD ZAR
```

...and I want to turn them into Ruby symbols for inclusion in Ruby source code as keys in a hash
whose values will be the display names of the currencies, e.g "Australian Dollar").
After copying this line to the clipboard, I could run this:

```
➜  ~   pbpaste | rexe -ml "split.map(&:downcase).map { |s| %Q{    #{s}: '',} }.join(%Q{\n})"
    aud: '',
    bgn: '',
    brl: '',
    # ...
```

If I add `| pbcopy` to the rexe command, then that output text would be copied into the clipboard instead of
displayed in the terminal, and I could then paste it in my editor.

Using the clipboard in manual operations is handy, but using it in automated scripts is a very bad idea, since
there is only one clipboard per user session. If you use the clipboard in an automated script you risk
an error situation if its content is changed by others, or, conversely, you could mess up another task
when you change the content of the clipboard. 

### Multiline Ruby Commands

Although `rexe` is cleanest with short one liners, you may want to use it to include nontrivial Ruby code
in your shell script as well. If you do this, you may need to add trailing backslashes to the lines of Ruby code.

What might not be so obvious is that you will often need to use semicolons. For example, here is an example without a semicolon:

```
➜  ~   cowsay hello | rexe -me "print %Q{\u001b[33m} \
puts to_a"

/Users/kbennett/.rvm/gems/ruby-2.6.0/gems/rexe-0.10.1/exe/rexe:256:in `eval':
   (eval):1: syntax error, unexpected tIDENTIFIER, expecting '}' (SyntaxError)
...new { print %Q{\u001b[33m} puts to_a }
...                           ^~~~
```

The shell combines all backslash terminated lines into a single line of text, so when the Ruby
interpreter sees your code, it's all in a single line. Adding the semicolon fixes the problem:

```
➜  ~   cowsay hello | rexe -me "print %Q{\u001b[33m}; \
puts to_a"
 _______
< hello >
 -------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```


### Clearing the Require and Load Lists

There may be times when you have specified a load or require on the command line
or in the `REXE_OPTIONS` environment variable,
but you want to override it for a single invocation. Here are your options:
 
1) Unspecify _all_ the requires or loads with the `-r!` and `-l!` command line options, respectively.

2) Unspecify individual requires or loads by preceding the name with `-`, e.g. `-r -rails`.
Array subtraction is used, so:

```
➜  ~   rexe -n -r rails,rails,rails,-rails -ga
```

...would show that the final `-rails` cancelled all the previous `rails` specifications.




### Clearing _All_ Options

You can also clear _all_ options specified up to a certain point in time with the _clear options_ option (`-c`).
This is especially useful if you have specified options in the `REXE_OPTIONS` environment variable, 
and want to ignore all of them.


### Comma Separated Requires and Loads

For consistency with the `ruby` interpreter, `rexe` supports requires with the `-r` option, but 
also allows grouping them together using commas:

```
                                    vvvvvvvvvvvvvvvvvvvvv
➜  ~   echo $EUR_RATES_JSON | rexe -r json,awesome_print 'ap JSON.parse(STDIN.read)'
                                    ^^^^^^^^^^^^^^^^^^^^^
```

Files loaded with the `-l` option are treated the same way.


### Beware of Configured Requires

Requiring gems and modules for _all_ invocations of `rexe` will make your commands simpler and more concise, but will be a waste of execution time if they are not needed. You can inspect the execution times to see just how much time is being wasted. For example, we can find out that rails takes about 0.63 seconds to load on my laptop by observing and comparing the execution times with and without the require (output has been abbreviated using the redirection and grep):

```
➜  ~   rexe -gy -r rails 123 2>&1 | grep duration
:duration_secs: 0.660138
➜  ~   rexe -gy          123 2>&1 | grep duration
:duration_secs: 0.027781
```
(For the above to work, the `rails` gem and its dependencies need to be installed.)


### Operating System Support

`rexe` has been tested successfully on Mac OS, Linux, and Windows Subsystem for Linux (WSL).
It is intended as a tool for the Unix shell, and, as such, no attempt is made to support
Windows non-Unix shells.


### More Examples

Here are some more examples to illustrate the use of `rexe`.

----

### Format Conversions -- JSON, YAML, AwesomePrint

You may work with YAML or JSON a lot and need to inspect files from time to time.
`rexe` makes it easy to convert from one format to another. For example, here's a
command to convert from `countries.yaml` to AwesomePrint:

```
cat countries.yaml | rexe -iy -oa -mb self
```

You can easily create a script that automates this. Using your favorite editor,
put this in a file:

```
cat $1 | rexe -iy -oa -mb self
```

I put mine in a file called `y2a` in my `~/bin` directory where I keep custom scripts like this.

Now you can call `y2a` to output any YAML file using AwesomePrint.

----

#### Outputting ENV

Output the contents of `ENV` using AwesomePrint [^3]:

```
➜  ~   rexe -oa ENV
{
...
                          "LANG" => "en_US.UTF-8",
                           "PWD" => "/Users/kbennett/work/rexe",
                         "SHELL" => "/bin/zsh",
...
}
```

----

#### Reformatting a Command's Output

Show disk space used/free on a Mac's main hard drive's main partition:

```
➜  ~   df -h | grep disk1s1 | rexe -ml \
"x = split; puts %Q{#{x[4]} Used: #{x[2]}, Avail: #{x[3]}}"
91% Used: 412Gi, Avail: 44Gi
```

(Note that `split` is equivalent to `self.split`, and because the `-ml` option is used, `self` is the line of text.

----

#### Formatting for Numeric Sort
    
Show the 3 longest file names of the current directory, with their lengths, in descending order:

```
➜  ~   ls  | rexe -ml "%Q{[%4d] %s} % [length, self]" | sort -r | head -3
[  50] Agoda_Booking_ID_9999999 49_–_RECEIPT_enclosed.pdf
[  40] 679a5c034994544aab4635ecbd50ab73-big.jpg
[  28] 2018-abc-2019-01-16-2340.zip
```

When you right align numbers using printf formatting, sorting the lines
alphabetically will result in sorting them numerically as well.

----

#### Print yellow (trust me!):

```
➜  ~   cowsay hello | rexe -me "print %Q{\u001b[33m}; puts to_a"
➜  ~     # or
➜  ~   cowsay hello | rexe -mb "print %Q{\u001b[33m}; puts self"
➜  ~     # or
➜  ~   cowsay hello | rexe "print %Q{\u001b[33m}; puts STDIN.read"
  _______
 < hello >
  -------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||`
```


----

----

#### More YouTube: Differentiating Success and Failure 

Let's go a little crazy with the YouTube example.
Let's have the video that loads be different for the success or failure
of the command.

If I put this in a load file (such as ~/.rexerc):

```
def play(piece_code)
  pieces = {
    hallelujah: "https://www.youtube.com/watch?v=IUZEtVbJT5c&t=0m20s",
    rick_roll:  "https://www.youtube.com/watch?v=dQw4w9WgXcQ&t=0m43s",
    valkyries:  "http://www.youtube.com/watch?v=P73Z6291Pt8&t=0m28s",
    wm_tell:    "https://www.youtube.com/watch?v=j3T8-aeOrbg",
  }
  `open #{Shellwords.escape(pieces.fetch(piece_code))}`
end


def play_result(success)
  play(success ? :hallelujah : :rick_roll)
end


def play_result_by_exit_code
  play_result(STDIN.read.chomp == '0')
end

```

Then when I issue a command that succeeds, the Hallelujah Chorus is played:

```
➜  ~   uname; echo $? | rexe play_result_by_exit_code
```

...but when the command fails, in this case, with an executable which is not found, it plays Rick Astley's
"Never Gonna Give You Up":

```
➜  ~   uuuuu; echo $? | rexe play_result_by_exit_code
```

----

#### Reformatting Source Code for Help Text

Another formatting example...I wanted to reformat this source code...

```
                                 'i' => Inspect
                                 'j' => JSON
                                 'J' => Pretty JSON
                                 'n' => No Output
                                 'p' => Puts (default)
                                 's' => to_s
                                 'y' => YAML
```

...into something more suitable for my help text.
Admittedly, the time it took to do this with rexe probably exceeded the time to do it manually,
but it was an interesting exercise and made it easy to try different formats. Here it is:

```
➜  ~   pbpaste | rexe -ml "sub(%q{'}, '-o').sub(%q{' =>}, %q{ })"
                                 -oi  Inspect
                                 -oj  JSON
                                 -oJ  Pretty JSON
                                 -on  No Output
                                 -op  Puts (default)
                                 -os  to_s
                                 -oy  YAML
```                                 
                                 

#### Reformatting Grep Output

I was recently asked to provide a schema for the data in my `rock_books` accounting gem. `rock_books` data is intended to be very small in size, and no data base is used. Instead, the input data is parsed on every run, and reports generated on demand. However, there are data structures (actually class instances) in memory at runtime, and their classes inherit from `Struct`.
 The definition lines look like this one:
 
```
class JournalEntry < Struct.new(:date, :acct_amounts, :doc_short_name, :description, :receipts)
```

The `grep` command line utility prepends each of these matches with a string like this:

```
lib/rock_books/documents/journal_entry.rb:
```

So this is what worked well for me:

```
➜  ~   grep Struct **/*.rb | grep -v OpenStruct | rexe -ml \
"a = \
 gsub('lib/rock_books/', '')\
.gsub('< Struct.new',    '')\
.gsub('; end',           '')\
.split('.rb:')\
.map(&:strip);\
\
%q{%-40s %-s} % [a[0] + %q{.rb}, a[1]]"
```

...which produced this output:

``` 
cmd_line/command_line_interface.rb       class Command (:min_string, :max_string, :action)
documents/book_set.rb                    class BookSet (:run_options, :chart_of_accounts, :journals)
documents/journal.rb                     class Entry (:date, :amount, :acct_amounts, :description)
documents/journal_entry.rb               class JournalEntry (:date, :acct_amounts, :doc_short_name, :description, :receipts)
documents/journal_entry_builder.rb       class JournalEntryBuilder (:journal_entry_context)
reports/report_context.rb                class ReportContext (:chart_of_accounts, :journals, :page_width)
types/account.rb                         class Account (:code, :type, :name)
types/account_type.rb                    class AccountType (:symbol, :singular_name, :plural_name)
types/acct_amount.rb                     class AcctAmount (:date, :code, :amount, :journal_entry_context)
types/journal_entry_context.rb           class JournalEntryContext (:journal, :linenum, :line)
``` 

Although there's a lot going on here, the vertical and horizontal alignments and spacing make the code
straightforward to follow. Here's what it does:

* grep the code base for `"Struct"`
* exclude references to `"OpenStruct"` with `grep -v`
* remove unwanted text with `gsub`
* split the line into 1) a filespec relative to `lib/rockbooks`, and 2) the class definition
* strip unwanted space because that will mess up the horizontal alignment of the output.
* use C-style printf formatting to align the text into two columns

 
### Conclusion

`rexe` is not revolutionary technology, it's just plumbing that removes parsing,
formatting, and low level
configuration from your command line so that you can focus on the high level
task at hand.

When we consider a new piece of software, we usually think "what would this be
helpful with now?". However, for me, the power of `rexe` is not so much what I can do
with it in a single use case now, but rather what will I be able to do with it over time
as I accumulate more experience and expertise.

I suggest starting to use `rexe` even for modest improvements in workflow, even
if it doesn't seem compelling. There's a good chance that as you use it over
time, new ideas will come to you and the workflow improvements will increase
exponentially.

A word of caution though -- 
the complexity and difficulty of sharing your `rexe` scripts across systems
will be proportional to the extent to which you use environment variables
and loaded files for configuration and shared code.
Be responsible and disciplined in making this configuration and code as clean and organized as possible.

----

#### Footnotes

[^1]: `rexe` is an embellishment of the minimal but excellent `rb` script at
https://github.com/thisredone/rb. I started using `rb` and thought of lots of
other features I would like to have, so I started working on `rexe`.

[^2]: Making this truly OS-portable is a lot more complex than it looks on the surface.
On Linux, `xdg-open` may not be installed by default. Also, Windows Subsystem for Linux (WSL)
out of the box is not able to launch graphical applications.

Here is a _start_ at a method that opens a resource portably across operating systems:

```ruby
  def open_resource(resource_identifier)
    command = case (`uname`.chomp)
    when 'Darwin'
      'open'
    when 'Linux'
      'xdg-open'
    else
      'start'
    end

    `#{command} #{resource_identifier}`
  end
```

[^3]: It is an interesting quirk of the Ruby language that `ENV.to_s` returns `"ENV"` and not the contents of the `ENV` object. As a result, most of the other output formats will return some form of `"ENV"`. You can handle this by specifying `ENV.to_h`.
