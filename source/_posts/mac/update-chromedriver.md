---
title: Mac 更新chromedriver的脚本
categories: 
  - MacOs
tags:
  - Mac
  - chromedriver
  - selenium
---


## 背景
我们在做自动化的时候的,特别是在mac上会遇到chrome浏览器自动升级的情况,而且还挺频繁的, 所以我们需要更新的chromedriver的版本, 不然就会无法启动chrome

## 解决文案  
### 环境变量下
写个shell的脚本来自动升级chromedriver的版本, 当然也可以用一些node的包来升级到最新或是指定driver的版本, 但是我查了一下, 基本上没有自动获取浏览器版本并升级相应的版本.  
花点时间自己写个适合自己的脚本, 不求人

``` sh
chrome=`/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --version`
cv=`echo ${chrome} | sed -e 's/.* //' -e 's/\..*//'`
echo current chrome browser version: ${cv}
path=`which chromedriver`
echo chrome driver path: ${path}
dv=`chromedriver -v | sed  -e 's/\..*//' -e 's/.*r //'`
echo current chrome driver version: $dv

if [ ${#path} \< 1 ] || [ $dv \< `expr $cv - 1` ] || [ $dv \> `expr $cv + 1` ]; then
echo need to update driver!
if [ ${#path} \< 1 ]; then
echo No chromedriver
path="/usr/local/bin/"
else
path=`echo ${path} | sed 's/chromedriver//'`
fi

LATEST_VERSION=$(curl -s https://chromedriver.storage.googleapis.com/LATEST_RELEASE_${cv}) && curl --output /tmp/chromedriver.zip https://chromedriver.storage.googleapis.com/$LATEST_VERSION/chromedriver_mac64.zip && sudo unzip /tmp/chromedriver.zip chromedriver -d ${path};
else
echo no need to update driver.
fi
chromedriver -v 
echo 'finished'
```

### selenium-standalone
如果你是用的[selenium-standalone](https://www.npmjs.com/package/selenium-standalone),那么有时候这个包的收录的driver版本的不够新. 用以下代码

``` sh
chrome=$(/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --version)
cv=$(echo ${chrome} | sed -e 's/.* //' -e 's/\..*//')
echo current chrome browser version: ${cv}
LATEST_VERSION=$(curl -s https://chromedriver.storage.googleapis.com/LATEST_RELEASE_${cv}) && selenium-standalone install --drivers.chromiumedge.version=${LATEST_VERSION};
echo 'finished'

```

## 现有方案
NPM上已经有了[ChromeDriver](https://www.npmjs.com/package/chromedriver)现有方案.
> npm install -g chromedriver --detect_chromedriver_version

可用这个下载到对应的node项目下. 可惜没有提供指定output路径.
 
