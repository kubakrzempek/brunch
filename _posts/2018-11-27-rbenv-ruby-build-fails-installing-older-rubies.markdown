# rbenv ruby-build fails installing older rubies

From time to time I'm required to install some older version of Ruby (by older I mean anything lower than 2.4), which happens to cause some issues in Ubuntu. Oftentimes it ends with cryptic:

```bash
Inspect or clean up the working tree at /tmp/ruby-build.20181127135105.18707
Results logged to /tmp/ruby-build.20181127135105.18707.log

Last 10 log lines:
compiling ../.././ext/psych/yaml/scanner.c
compiling ../.././ext/psych/yaml/api.c
compiling ../.././ext/psych/yaml/writer.c
installing default psych libraries
linking shared-object nkf.so
make[2]: Leaving directory '/tmp/ruby-build.20181127135105.18707/ruby-2.3.1/ext/nkf'
linking shared-object psych.so
make[2]: Leaving directory '/tmp/ruby-build.20181127135105.18707/ruby-2.3.1/ext/psych'
make[1]: Leaving directory '/tmp/ruby-build.20181127135105.18707/ruby-2.3.1'
make: *** [uncommon.mk:203: build-ext] Error 2
```

That's because in [rbenv/ruby-build](https://github.com/rbenv/ruby-build) older (<2.4) rubies require `libssl v1.0`, while newer want `libssl v1.1` and up. The two `libssl` packages conflict with each other, thus one has to switch between them in order to install rubies from both sides of the boundary. Also, to make things funnier the older rubies tends to fail with newer `gcc` (v7) and want to have v6 on board. So, usually, the install process looks like that:

```bash
# for ruby <2.4
apt install libssl1.0-dev
CC=$(which gcc-6) rbenv install `version`

# for ruby >= 2.4
apt install libssl-dev
rbenv install `version`
```

It's worth noting that if you are going to install any gems that require building native extension you also have to have a matching version of `libssl`
