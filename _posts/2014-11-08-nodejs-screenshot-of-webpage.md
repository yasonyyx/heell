---
layout: post
title: node.js网页截图
---

最近有一个需求，是根据配置，用程序生成类似下面的图片：

![screenshto](http://wong2.qiniudn.com/2014-11-08-screenshot.png)

首先图片有一个背景图，然后有title和一个按钮，按钮里有文字。背景图和文字都是写在配置文件里的。
难点在于，文字和按钮都是有特殊效果的（渐变，阴影，描边）。

一开始，我尝试用Python+Pillow来绘制图片，但是发现非常困难，因为Pillow（PIL）提供的接口都比较底层（low level），比如写文字需要指定 (x, y) 坐标，要实现居中都得先计算出文字的宽度然后再算坐标。 渐变阴影之类的效果，就更困难了。

然后我突然想到，CSS里不是可以很简单的设置渐变、阴影吗，居中什么的更是家常便饭。于是有了一个新思路：根据配置生成HTML文件，然后对这些网页进行截图，就能得到想要的图片了。

#### PhantomJS/Pageres

在用jade完成了根据配置生成HTML文件后，开始了网页截图的实现。

首先想到的是 [PhantomJS](http://phantomjs.org/)，之前用过PhantomJS来对网页截图，但是我不太喜欢 [phantomjs-node](https://github.com/sgentle/phantomjs-node)这每步都是回调的用法，于是找了下有没有调用PhantomJS截图的库，就找到了[pageres](https://github.com/sindresorhus/pageres)，代码如下：

    function takeScreenshot(htmlPath, outPath, size, done) {
      var pageres = new Pageres()
        .src(htmlPath, [size])
	    .run(function(err, items) {
	      if (err) {
	        done(err);
	      } else {
	        var f = fs.createWriteStream(outPath).on('finish', done);
	        items[0].pipe(f);
	      }
	    });
	}
	
#### webkit2png
	
pageres的方案试用之后，发现有时，生成的图片效果和浏览器里看到的不同（可能因为用到了 -webkit-text-stroke 这个比较新的CSS属性），怀疑是因为 PhantomJS 所用的 Webkit 的版本相对较老。折腾了一下未果之后，想到了直接调用系统里的Webkit进行截图的 `webkit2png`。

[webkit2png](http://www.paulhammond.org/webkit2png/) 是一个Python写的网页截图脚本，通过调用OSX的接口来通过Webkit对网页进行截图。

找了一下，没有找到 webkit2png 的nodejs绑定，于是只好通过`child_process`来执行了，简单包装了一个 webki2png.js如下：

    var util = require('util');
	var exec = require('child_process').exec;
	
	require('shelljs/global');
	
	module.exports.isSupported = function() {
	  return !!which('webkit2png');
	};
	
	module.exports.capture = function (url, options, callback) {
	
	  if (!callback && typeof options === "function") {
	    callback = options;
	    options = {};
	  }
	
	  var command = ['webkit2png'];
	  var arg = null;
	
	  Object.keys(options).forEach(function(key) {
	    var value = options[key]
	    if (value === true) {
	      arg = util.format('--%s', key);
	    } else {
	      arg = util.format('--%s=%s', key, options[key]);
	    }
	    command.push(arg);
	  });
	  command.push(url);
	
	  command = command.join(' ');
	  var child = exec(command, function(error, stdout, stderr) {
	    callback && callback(error);
	  });
	
	};
	
#### ChromeDriver

换上 webkit2png 之后，之前的问题解决了，但是又出现了新的问题：对泰文生成的图片效果不好，具体来说，相同的网页，Chrome显示很完美，Safari很差劲，webkit2png的效果呢，介于二者之间。

为了更完美的效果，开始寻找控制Chrome截图的方案，最后找到了ChromeDriver。

首先要知道什么是 WebDriver，我们知道[Selenium](http://www.seleniumhq.org/)是一个网站自动测试工具，通过它可以编程控制浏览器进行一些操作，WebDriver就是抽象出的一层控制浏览器的API，ChromeDriver则是针对Chrome的WebDriver API的实现。

首先需要在ChromeDriver的官网下载： <https://sites.google.com/a/chromium.org/chromedriver/>
解压后会有一个可执行文件 `chromedriver`，执行它，就会在默认的9515端口启动一个Webdriver服务器。然后借助 `wd` 这个nodejs的WebDriver库，就可以控制Chrome进行截图了，简单封装了 `chrome2png.js` 如下：

	var fs = require('fs');
	var path = require('path');
	var wd = require('wd');
	var gm = require('gm');
	
	var browser = wd.promiseChainRemote('http://localhost:9515/');
	
	module.exports.capture = function(htmlPath, outPath, size, callback) {
	  var url = 'file://' + path.resolve(htmlPath);
	  browser
	    .init({browserName:'chrome'})
	    .get(url)
	    .takeScreenshot()
	    .then(function(base64Data) {
	      fs.writeFileSync(outPath, base64Data, 'base64');
	      gm(outPath).crop(size.width, size.height, 0, 0).write(outPath, callback);
	    })
	    .fail(callback)
	    .fin(function() {
	      return browser.quit();
	    })
	    .done();
	};
		
生成的效果非常完美。唯一的缺点是，在截图过程中，浏览器窗口会弹出，不过在我这个需求下是可以忍受的。