# MacOS Required Packages

## XCode

* From the App Store

## Brew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Node 14

```bash
brew install node@14
```

* After installing,
  * Create the file `/etc/paths.d/node`
  * ... with the content `/usr/local/opt/node@14/bin`

## Yarn

```bash
npm install --global yarn
```

## Sentry

```bash
curl -sL https://sentry.io/get-cli/ | bash
```

## Cocoa Pods

```bash
sudo gem install cocoapods
```
