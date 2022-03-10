## Mac把npm换成cnpm
### NPM设置淘宝镜像
1. 查询当前配置的镜像
npm get registry 
2. 设置成淘宝镜像
npm config set registry http://registry.npm.taobao.org/
3. 换成原来的
npm config set registry https://registry.npmjs.org/
### Yarn js包管理器设置淘宝镜像
1. 查询当前配置的镜像
yarn config get registry
2. 设置成淘宝镜像
yarn config set registry http://registry.npm.taobao.org/
3. 如果报错ERROR: Failed to download Chromium r722234! Set "PUPPETEER_SKIP_CHROMIUM_DOWNLOAD" env variable to skip download.
npm config set puppeteer_download_host=https://npm.taobao.org/mirrors
npm i puppeteer
4. 或者
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm i puppeteer

## 