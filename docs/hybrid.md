Automating hybrid apps
======================

One of the core principles of Appium is that you shouldn't have to change your app to test it. In line with that methodology, it is possible to test hybrid web apps (e.g., the "UIWebView" elements in an iOS app) the same* way you can with Selenium for web apps. There is a bit of technical complexity required so that Appium knows whether you want to automate the native aspects of the app or the web views, but thankfully, we can stay within the WebDriver protocol for everything.

Here are the steps required to talk to a web view in your Appium test:

1.  Navigate to a portion of your app where a web view is active
1.  Call [GET session/:sessionId/window_handles](http://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/window_handles)
1.  This returns a list of web view ids we can access
1.  Call [POST session/:sessionId/window](http://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/window) with the id of the web view you want to access
1.  (This puts your Appium session into a mode where all commands are interpreted as being intended for automating the web view, rather than the native portion of the app. For example, if you run getElementByTagName, it will operate on the DOM of the web view, rather than return UIAElements. Of course, certain WebDriver methods only make sense in one context or another, so in the wrong context you will receive an error message).
1.  To stop automating in the web view context and go back to automating the native portion of the app, simply call `"mobile: leaveWebView"` with execute_script to leave the web frame.

## Execution against a real iOS device
To interrogate and interact with a web view appium establishes a connection using a remote debugger. When executing the examples below against a simulator this connection can be established directly as the simulator and the appium server are on the same machine. When executing against a real device appium is unable to access the web view directly. Therefore the connection has to be established through the USB lead. To establish this connection we use the [ios-webkit-debugger-proxy](https://github.com/google/ios-webkit-debug-proxy).

To install the latest tagged version of the ios-webkit-debug-proxy using brew, run the following commands in the terminal:
``` bash
# The first command is only required if you don't have brew installed.
> ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go)"
> brew update
> brew install ios-webkit-debug-proxy
```

You can also install the latest proxy by cloning it from git and installing it yourself:
``` bash
# Please be aware that this will install the proxy with the latest code (and not a tagged version).
> git clone https://github.com/google/ios-webkit-debug-proxy.git
> cd ios-webkit-debug-proxy
> ./autogen.sh
> ./configure
> make
> sudo make install
```

Once installed you can start the proxy with the following command:
``` bash
# Change the udid to be the udid of the attached device and make sure to set the port to 27753 
# as that is the port the remote-debugger uses.
> ios_webkit_debug_proxy -c 0e4b2f612b65e98c1d07d22ee08678130d345429:27753 -d
``` 

<b>NOTE:</b> the proxy requires the <b>"web inspector"</b> to be turned on to allow a connection to be established. Turn it on by going to <b> settings > safari > advanced </b>. Please be aware that the web inspector was <b>added as part of iOS 6</b> and was not available previously.

## Wd.js Code example

```js
  // assuming we have an initialized `driver` object working on the UICatalog app
  driver.elementByName('Web, Use of UIWebView', function(err, el) { // find button to nav to view
    el.click(function(err) { // nav to UIWebView
      driver.windowHandles(function(err, handles) { // get list of available views
        driver.window(handles[0], function(err) { // choose the only available view
          driver.elementsByCss('.some-class', function(err, els) { // get webpage elements by css
            els.length.should.be.above(0); // there should be some!
            els[0].text(function(elText) { // get text of the first element
              elText.should.eql("My very own text"); // it should be extremely personal and awesome
              driver.execute("mobile: leaveWebView", function(err) { // leave webview context
                // do more native stuff here if we want
                driver.quit(); // stop webdrivage
              });
            });
          });
        });
      });
    });
  });
```

* For the full context, see [this node example](https://github.com/appium/appium/blob/master/sample-code/examples/node/hybrid.js)
* *we're working on filling out the methods available in web view contexts. [Join us in our quest!](http://appium.io/get-involved.html)

## Wd.java Code example

```java
  //setup the web driver and launch the webview app.
  DesiredCapabilities desiredCapabilities = new DesiredCapabilities();
  desiredCapabilities.setCapability("device", "iPhone Simulator");
  desiredCapabilities.setCapability("app", "http://appium.s3.amazonaws.com/WebViewApp6.0.app.zip");  
  URL url = new URL("http://127.0.0.1:4723/wd/hub");
  RemoteWebDriver remoteWebDriver = new RemoteWebDriver(url, desiredCapabilities);
  
  //switch to the latest web view
  for(String winHandle : remoteWebDriver.getWindowHandles()){
    remoteWebDriver.switchTo().window(winHandle);
  }
  
  //Interact with the elements on the guinea-pig page using id.
  WebElement div = remoteWebDriver.findElement(By.id("i_am_an_id"));
  Assert.assertEquals("I am a div", div.getText()); //check the text retrieved matches expected value
  remoteWebDriver.findElement(By.id("comments")).sendKeys("My comment"); //populate the comments field by id.
  
  //leave the webview to go back to native app.
  remoteWebDriver.executeScript("mobile: leaveWebView");      
  
  //close the app.
  remoteWebDriver.quit();
```

