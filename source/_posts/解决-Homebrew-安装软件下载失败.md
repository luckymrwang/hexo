title: 解决 Homebrew 安装软件下载失败
date: 2020-09-28 08:57:35
tags: [Brew]
---

>当我们使用 Homebrew 安装软件时，由于一些特殊原因会出现软件包下载失败的情况。这种还很常见，我们没法改变环境，但却可以取巧的解决，那就是利用 Homebrew 缓存的特性，手动预先下载软件。

<!-- more -->

###方法一: 手动下载软件包到缓存目录

以安装 `Dart` 为例:

```sh
$ brew install dart
==> Installing dart from dart-lang/dart
==> Downloading https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip

curl: (56) LibreSSL SSL_read: SSL_ERROR_SYSCALL, errno 54
Error: An exception occurred within a child process:
  DownloadError: Failed to download resource "dart"
Download failed: https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
```

无法成功下载对应的软件包，但 `Homebrew` 会告知软件的下载地址:

```sh
Download failed: https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
```

于是，我们可以手动下载这个软件。然后我们获取缓存目录:

```sh
$ brew --cache
/Users/shockerli/Library/Caches/Homebrew
```

把刚下载好的软件包拷贝到这个目录下:

```sh
$ cp ~/Downloads/dartsdk-macos-x64-release.zip /Users/shockerli/Library/Caches/Homebrew/
```

我们再执行安装命令，不出意外，那么恭喜你，成功解决了问题。

但凡是也就有意外，不幸的你跟我一样，发现还是报错了:

```sh
$ brew install dart
==> Installing dart from dart-lang/dart
==> Downloading https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos

curl: (56) LibreSSL SSL_read: SSL_ERROR_SYSCALL, errno 54
Error: An exception occurred within a child process:
  DownloadError: Failed to download resource "dart"
Download failed: https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
```

那该怎么解决？我们给命令加个 -v 打印命令的详细日志看看:

```sh
brew install dart -v
==> Installing dart from dart-lang/dart
/usr/bin/sandbox-exec -f /private/tmp/homebrew20190810-60798-mlb2s.sb nice /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby -W0 -I /usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-cobertura-1.3.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-macho-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-rspec-1.35.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-performance-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-0.74.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unicode-display_width-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-progressbar-1.10.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-wait-0.0.9/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-retry-0.6.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-its-1.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-3.8.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-mocks-3.8.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-expectations-3.8.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-core-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-support-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ronn-0.7.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rdiscount-2.2.0.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/rdiscount-2.2.0.1:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rainbow-3.0.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/plist-3.5.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parser-2.6.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel_tests-2.29.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel-1.17.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mustache-1.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mechanize-2.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/webrobots-0.1.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ntlm-http-0.1.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/nokogiri-1.10.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/nokogiri-1.10.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mini_portile2-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-persistent-3.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-digest_auth-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-3.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-data-3.2019.0331/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/jaro_winkler-1.5.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/jaro_winkler-1.5.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/http-cookie-1.0.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/hpricot-0.8.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/hpricot-0.8.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/domain_name-0.5.20190701/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf-0.1.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf_ext-0.0.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/unf_ext-0.0.7.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/diff-lcs-1.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/coveralls-0.8.23/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thor-0.20.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/term-ansicolor-1.7.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tins-1.21.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-0.16.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-html-0.10.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/docile-1.3.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/json-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/json-2.2.0:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/connection_pool-2.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/backports-3.15.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ast-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/activesupport-5.2.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tzinfo-1.2.5/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thread_safe-0.3.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/minitest-5.11.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/i18n-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/concurrent-ruby-1.1.5/lib:/Library/Ruby/Site/2.3.0:/Library/Ruby/Site/2.3.0/x86_64-darwin18:/Library/Ruby/Site/2.3.0/universal-darwin18:/Library/Ruby/Site:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/universal-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/universal-darwin18:/usr/local/Homebrew/Library/Homebrew -- /usr/local/Homebrew/Library/Homebrew/build.rb /usr/local/Homebrew/Library/Taps/dart-lang/homebrew-dart/dart.rb --verbose

==> Downloading https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
/usr/bin/curl -q --show-error --user-agent Homebrew/2.1.9-21-g625a780\ \(Macintosh\;\ Intel\ Mac\ OS\ X\ 10.14.2\)\ curl/7.54.0 --fail --location --remote-time --continue-at 0 --output /Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip.incomplete https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:43 --:--:--     0
curl: (56) LibreSSL SSL_read: SSL_ERROR_SYSCALL, errno 60
Error: An exception occurred within a child process:
  DownloadError: Failed to download resource "dart"
Download failed: https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
```

注意到这条信息:

```sh
/usr/bin/curl -q --show-error --user-agent Homebrew/2.1.9-21-g625a780\ \(Macintosh\;\ Intel\ Mac\ OS\ X\ 10.14.2\)\ curl/7.54.0 --fail --location --remote-time --continue-at 0 --output /Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip.incomplete https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip
```

我们看到 `Homebrew` 下载 `dart` 的缓存地址为:

```sh
/Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip.incomplete
```

`XXX.incomplete` 表示下载未完成，但这是 `Homebrew` 期望的下载文件路径。

那我们就将下载的软件包拷贝到这个路径，并去除 `.incomplete`:

```sh
mv ~/Downloads/dartsdk-macos-x64-release.zip /Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip
```

此时我们再执行安装命令:

```sh
brew install dart -v
==> Installing dart from dart-lang/dart
/usr/bin/sandbox-exec -f /private/tmp/homebrew20190811-62563-1rnamem.sb nice /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby -W0 -I /usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-cobertura-1.3.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-macho-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-rspec-1.35.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-performance-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-0.74.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unicode-display_width-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-progressbar-1.10.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-wait-0.0.9/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-retry-0.6.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-its-1.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-3.8.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-mocks-3.8.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-expectations-3.8.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-core-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-support-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ronn-0.7.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rdiscount-2.2.0.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/rdiscount-2.2.0.1:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rainbow-3.0.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/plist-3.5.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parser-2.6.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel_tests-2.29.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel-1.17.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mustache-1.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mechanize-2.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/webrobots-0.1.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ntlm-http-0.1.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/nokogiri-1.10.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/nokogiri-1.10.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mini_portile2-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-persistent-3.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-digest_auth-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-3.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-data-3.2019.0331/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/jaro_winkler-1.5.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/jaro_winkler-1.5.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/http-cookie-1.0.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/hpricot-0.8.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/hpricot-0.8.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/domain_name-0.5.20190701/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf-0.1.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf_ext-0.0.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/unf_ext-0.0.7.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/diff-lcs-1.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/coveralls-0.8.23/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thor-0.20.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/term-ansicolor-1.7.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tins-1.21.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-0.16.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-html-0.10.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/docile-1.3.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/json-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/json-2.2.0:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/connection_pool-2.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/backports-3.15.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ast-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/activesupport-5.2.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tzinfo-1.2.5/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thread_safe-0.3.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/minitest-5.11.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/i18n-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/concurrent-ruby-1.1.5/lib:/Library/Ruby/Site/2.3.0:/Library/Ruby/Site/2.3.0/x86_64-darwin18:/Library/Ruby/Site/2.3.0/universal-darwin18:/Library/Ruby/Site:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/universal-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/universal-darwin18:/usr/local/Homebrew/Library/Homebrew -- /usr/local/Homebrew/Library/Homebrew/build.rb /usr/local/Homebrew/Library/Taps/dart-lang/homebrew-dart/dart.rb --verbose
==> Downloading https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip


Already downloaded: /Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip
==> Verifying 4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip checksum
unzip /Users/shockerli/Library/Caches/Homebrew/downloads/4d2412a5d84521393e0e1ecdce0662569e13c2c47762093a760939fa9dd4a917--dartsdk-macos-x64-release.zip -d /private/tmp/d20190811-62564-1me3qph
cp -pR /private/tmp/d20190811-62564-1me3qph/dart-sdk/. /private/tmp/dart-20190811-62564-hu8qrn/dart-sdk
chmod -Rf +w /private/tmp/d20190811-62564-1me3qph
==> Cleaning
==> Finishing up
ln -s ../Cellar/dart/2.4.1/bin/dart dart
ln -s ../Cellar/dart/2.4.1/bin/dart2aot dart2aot
ln -s ../Cellar/dart/2.4.1/bin/dart2js dart2js
ln -s ../Cellar/dart/2.4.1/bin/dartanalyzer dartanalyzer
ln -s ../Cellar/dart/2.4.1/bin/dartaotruntime dartaotruntime
ln -s ../Cellar/dart/2.4.1/bin/dartdevc dartdevc
ln -s ../Cellar/dart/2.4.1/bin/dartdoc dartdoc
ln -s ../Cellar/dart/2.4.1/bin/dartfmt dartfmt
ln -s ../Cellar/dart/2.4.1/bin/pub pub
/usr/bin/sandbox-exec -f /private/tmp/homebrew20190811-62996-w9cwbv.sb nice /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby -W0 -I /usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-cobertura-1.3.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-macho-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-rspec-1.35.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-performance-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rubocop-0.74.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unicode-display_width-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ruby-progressbar-1.10.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-wait-0.0.9/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-retry-0.6.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-its-1.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-3.8.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-mocks-3.8.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-expectations-3.8.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-core-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rspec-support-3.8.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ronn-0.7.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rdiscount-2.2.0.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/rdiscount-2.2.0.1:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/rainbow-3.0.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/plist-3.5.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parser-2.6.3.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel_tests-2.29.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/parallel-1.17.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mustache-1.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mechanize-2.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/webrobots-0.1.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ntlm-http-0.1.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/nokogiri-1.10.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/nokogiri-1.10.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mini_portile2-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-persistent-3.1.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/net-http-digest_auth-1.4.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-3.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/mime-types-data-3.2019.0331/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/jaro_winkler-1.5.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/jaro_winkler-1.5.3:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/http-cookie-1.0.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/hpricot-0.8.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/hpricot-0.8.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/domain_name-0.5.20190701/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf-0.1.4/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/unf_ext-0.0.7.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/unf_ext-0.0.7.6:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/diff-lcs-1.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/coveralls-0.8.23/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thor-0.20.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/term-ansicolor-1.7.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tins-1.21.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-0.16.1/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/simplecov-html-0.10.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/docile-1.3.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/json-2.2.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/extensions/universal-darwin-18/2.3.0/json-2.2.0:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/connection_pool-2.2.2/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/backports-3.15.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/ast-2.4.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/activesupport-5.2.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/tzinfo-1.2.5/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/thread_safe-0.3.6/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/minitest-5.11.3/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/i18n-1.6.0/lib:/usr/local/Homebrew/Library/Homebrew/vendor/bundle/bundler/../ruby/2.3.0/gems/concurrent-ruby-1.1.5/lib:/Library/Ruby/Site/2.3.0:/Library/Ruby/Site/2.3.0/x86_64-darwin18:/Library/Ruby/Site/2.3.0/universal-darwin18:/Library/Ruby/Site:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby/2.3.0/universal-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/vendor_ruby:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/x86_64-darwin18:/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/ruby/2.3.0/universal-darwin18:/usr/local/Homebrew/Library/Homebrew -- /usr/local/Homebrew/Library/Homebrew/postinstall.rb /usr/local/Homebrew/Library/Taps/dart-lang/homebrew-dart/dart.rb -v
==> Caveats
Please note the path to the Dart SDK:
  /usr/local/opt/dart/libexec
==> Summary
🍺  /usr/local/Cellar/dart/2.4.1: 387 files, 344.4MB, built in 3 minutes 11 seconds
```

`Homebrew` 成功用到了缓存并安装成功~祝贺你😆

###Homebrew 成功用到了缓存并安装成功~祝贺你😆

具体可见 `brew edit` 命令

比如修改 `dart` 包，`brew edit dart`:

```ruby
class Dart < Formula
  desc "The Dart SDK"
  homepage "https://www.dartlang.org/"

  version "2.4.1"
  if Hardware::CPU.is_64_bit?
    url "https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-x64-release.zip"
    sha256 "62006127bd3acd1b7eb2e4fc7baed061eb19b80c4ba4af481db5244a081fff3e"
  else
    url "https://storage.googleapis.com/dart-archive/channels/stable/release/2.4.1/sdk/dartsdk-macos-ia32-release.zip"
    sha256 "3591578902f3b3ee155aa90bf893f3d0b50fd12567454a8f980440fa8dd1ff23"
  end

  devel do
    version "2.5.0-dev.1.0"
    if Hardware::CPU.is_64_bit?
      url "https://storage.googleapis.com/dart-archive/channels/dev/release/2.5.0-dev.1.0/sdk/dartsdk-macos-x64-release.zip"
      sha256 "2fc3967437e8a3e2f5ee9d0abbf15ebf5823fdc6a648ee58a72bed550f93281f"
    else
      url "https://storage.googleapis.com/dart-archive/channels/dev/release/2.5.0-dev.1.0/sdk/dartsdk-macos-ia32-release.zip"
      sha256 "6c0b7c6719ded33eb9ada8a6f52d80b3fe94ab43e008bb432ef8f7fd4997145b"
    end
  end

  def install
    libexec.install Dir["*"]
    bin.install_symlink "#{libexec}/bin/dart"
    bin.write_exec_script Dir["#{libexec}/bin/{pub,dart?*}"]
  end

  def shim_script(target)
    <<~EOS
      #!/usr/bin/env bash
      exec "#{prefix}/#{target}" "$@"
    EOS
  end

  def caveats; <<~EOS
    Please note the path to the Dart SDK:
      #{opt_libexec}
    EOS
  end

  test do
    (testpath/"sample.dart").write <<~EOS
      void main() {
        print(r"test message");
      }
    EOS

    assert_equal "test message\n", shell_output("#{bin}/dart sample.dart")
  end
end
```

我们把 `tap` 里包对应的 `url` 给改成可用的地址即可。当然这种方法的前提是有可用的下载地址，比如清华大学、中科大等国内镜像源提供的加速地址，否则依然无效。