1. 安装node 注意版本不能太高 参考:https://nodejs.org/zh-cn/download/package-manager
```java
# layouts.download.codeBox.installsNvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# layouts.download.codeBox.downloadAndInstallNodejsRestartTerminal
nvm install 12
# layouts.download.codeBox.verifiesRightNodejsVersion
node -v # layouts.download.codeBox.shouldPrint
# layouts.download.codeBox.verifiesRightNpmVersion
npm -v # layouts.download.codeBox.shouldPrint
```
2. 安装 hexo
```java
npm install -g hexo-cli
```

3. hexo命令
```java
hexo g #生成文件
hexo s #本地部署
hexo d #部署到GitHub
```
