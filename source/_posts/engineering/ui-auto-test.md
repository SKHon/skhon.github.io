---
title: 【稳定性建设】UI自动化测试(一)
categories: 工程化建设
tags: [Engineering]
comments: true
copyright: true
toc: true
---

## 原理
通过Puppeteer模拟用户操作

## Puppeteer介绍
你可以在浏览器中手动完成的大部分事情都可以使用 Puppeteer 完成！你可以从以下几个示例开始：
    
- 生成页面的截图和PDF。
- 抓取SPA并生成预先呈现的内容（即“SSR”）。
- 从网站抓取你需要的内容。
- 自动表单提交，UI测试，键盘输入等
- 创建一个最新的自动化测试环境。使用最新的JavaScript和浏览器功能，直接在最新版本的Chrome中运行测试。
- 捕获您的网站的时间线跟踪，以帮助诊断性能问题。

还有一些可用于工程化相关的建设：
- 自动化工具 如自动提交表单，自动下载
- 自动化 UI 测试 如记录下正确 DOM 结构或截图，然后自动执行指定操作后，检查 DOM 结构或截图是否匹配(UI 断言)
- 定时监控工具 如定时截图发周报，或定时巡查重要业务路径下的页面是否处于可用状态，配合邮件告警

## Puppeteer文档
个人感觉，这个[文档](https://zhaoqize.github.io/puppeteer-api-zh_CN/#?product=Puppeteer&version=v10.4.0&show=outline)比较适合我。里面api比较详细，还有写小demo。

## 为什么要进行UI自动化测试

业务的更新迭代频繁，传统测试大部分都还是手工、肉眼的模式来进行，无法满足产品敏捷开发、快速迭代的需求。而UI自动化能让全功能的回归变得简单，释放纯手工测试的人力资源，并且回归测试能够覆盖到所有的逻辑场景，这对测试的效率，以及整个开发流程的效率都是很大的提升，并且能够规避很多人的主观和客观因素导致的漏测或者疏忽。
其他测试方式的局限性：

**单元测试（Unit Testing）**
事实上，单元测试确实能够帮助我们发现大部分的问题，但是在复杂的前端交互中，单纯的单元测试并不能真实地反映用户操作的路径，而单元测试一般的场景是测试一系列的功能集合。

**快照测试（Snapshot Testing）**
DOM结构并不能完全反映页面的视觉效果，DOM结构不变并不完全等于样式不变。此外，大多数工具都是React专用，非React应用基本不支持。

> 很多人认为，UI总是频繁的变动，导致测试用例维护成本高，性价比低，因此UI自动化测试比较适合场景稳定的业务。其实不是，这里的UI不仅仅指的是视觉，更多的是业务逻辑。UI可以多变，但业务逻辑一定是趋于稳定的，尤其是核心业务，想一想用户得多辛苦才能适应这种业务逻辑频繁变更的产品啊。

## 实战

### 第1步：确定功能测试方案

我们将在以下情况下逐步展示执行Puppeteer自动化的方法– 
·启动Web浏览器。
·调用Amazon Web应用程序。
- 搜索书籍“ Testing Book”。
- 从结果中将书添加到购物车中。
- 打开购物车，然后检查购物车中是否有书籍。
- 捕获屏幕并关闭浏览器。

### Step2＃安装Puppeteer并创建测试用例

在特定文件夹中创建一个空的javascript文件作为“ sample_script.js”。 在这里，我们将根文件夹视为SampleProject。 要安装Puppeteer，我们将使用命令–“ npm install puppeteer”。 根据网络速度，安装过程会花费一些时间。 它将下载大约350MB的数据。 安装后，将在示例Puppeteer项目根文件夹中创建包含不同的puppeteer组件和package-lock.json文件的node_modules文件夹。

### Step3＃捕获测试对象的标识属性
我们可以使用Chrome网络浏览器的开发者工具捕获标识属性。 分析不同的属性，例如id，name，XPath等，我们将选择可以在脚本执行任何操作中使用的正确属性。 在本“ Puppeteer自动化测试”教程中，我们将在脚本中使用XPath。 按照以下步骤获取XPATH或任何其他属性，

1. 打开“菜单->更多工具”下可用的开发人员工具，然后转到“元素”选项卡。

2. 使用 Finder 工具（单击 Elements 选项卡左上角可用的箭头图标），突出显示应用程序中的测试对象。 在这里，我们将检查搜索框。
<img src="/images/ui-test/ui1.png" >

3. 分析突出显示的源代码以识别期望的属性。 要获取测试对象的XPATH属性，请右键单击突出显示的部分，然后单击“复制->复制Xpath”以将XPATH属性复制到剪贴板中。
<img src="/images/ui-test/ui2.png" >

4. 现在，将Xpath粘贴到查找器文本框中，然后按Enter键以检查Xpath是否在唯一地标识对象。
<img src="/images/ui-test/ui3.png" >

5. 同样，我们还需要捕获另一个测试对象的标识属性。

### Step4＃Puppeteer自动化开发步骤

现在，我们已经了解了使功能场景自动化的基本技术步骤。 基于这些知识，我们可以通过下面的Puppeteer Automation测试案例。 最常用的类和方法的详细概述将在后续文章中进行解释。

```
/**
 * @name Amazon search
 */
const puppeteer = require('puppeteer');
const reportPath = 'C:\\LambdaGeeks\\puppteer_proj_sample\\output\\';
const screenshot = 'screen1.png';
// Used to export the file into a .docx file
try {
  (async () => {
    const browser = await puppeteer.launch({ headless: false });
    const pageNew = await browser.newPage()
    await pageNew.setViewport({ width: 1280, height: 800 });
    await pageNew.goto('https://www.amazon.in/');
    //Enter Search criteria
    let searchBox = await page.waitForXPath("//*[@id='twotabsearchtextbox']",{ visible: true });
    if (searchBox === null)
    {
        console.log('Amazon screen is not displayed');
    }
    else{       
        await searchBox.type("Testing Book");
        console.log('Search criteria has been entered');
    }       
    //Clicked on search button
    let btnSearch = await pageNew.waitForXPath("//*/input[@id='nav-search-submit-button']",{ visible: true });
    if (btnSearch === null)
    {
        console.log('Search button is not showing');
    }
    else{
        await btnSearch.click();
        console.log('Clicked on search button');
    }   
    //Click on specific search result
    let myBook = await pageNew.waitForXPath("//*[contains(text(),'Selenium Testing Tools Cookbook Second Edition')]",{ visible: true })
    if (myBook === null)
    {
        console.log('Book is not available');
    }
    else{
        await myBook.click();
        console.log('Click on specific book to order');
    }   
    // Identify if the new tab has opened
    const pageTarget = pageNew.target();
    const newTarget = await browser.waitForTarget(target => target.opener() === pageTarget);
    //get the new page object:
    const page2 = await newTarget.pageNew();    
    await page2.setViewport({ width: 1280, height: 800 });
     
    //Add to cart
    let addToCart = await page2.waitForXPath("//*/input[@id='add-to-cart-button']",{ visible: true });
    if (addToCart === null)
    {
        console.log('Add to cart button is not available');
    }
    else{
        console.log('Click on add to Cart button');
        await addToCart.click();        
    }       
    //Verify add to cart process    
    let successMessage = await page2.waitForXPath("//*[contains(text(),'Added to Cart')]",{ visible: true });
    if (successMessage === null)
    {
        console.log('Item is not added to cart');
    }
    else{
        console.log('Item is added to cart successfully');      
    }       
    // Capture no of cart
    let cartCount = await page2.waitForXPath("//*/span[@id='nav-cart-count']",{ visible: true});
    let value = await page2.evaluate(el => el.textContent, cartCount)
    console.log('Cart count: ' + value);
    cartCount.focus();
    await page2.screenshot({ path: screenshot });
     
    await pageNew.waitForTimeout(2000);    
    await page2.close();
    await pageNew.close();
    await browser.close();
  })()
} catch (err) {
  console.error(err)
}
```
### Step5＃木偶自动化测试执行
我们可以使用以下命令启动执行 节点sample_script.js 通过命令提示符。 在执行过程中，将打开Chromium浏览器并自动执行功能步骤，并存储最后一页的屏幕截图。 屏幕截图和控制台输出如下所示。
<img src="/images/ui-test/ui4.png" >