# Selenium 自动化测试实战

# 一、Selenium起步

## Web自动化测试
1. 什么是测试
- 通俗地讲，程序测试就是运行程序，并发现程序的错误。
- 专业地讲，验证软件的正确性、完整性、安全性和质量的过程
- 换言之就是找Bug

2. 测试的分类
按开发阶段划分：
```
单元测试
集成测试
系统测试
验收测试
```

按是否查看代码划分：
```
黑盒测试
白盒测试
灰盒测试
```

按是否运行划分：
```
静态测试
动态测试
```

按测试对象划分：
```
性能测试
安全测试
兼容性测试
文档测试
用户体验测试
业务测试
界面测试
安装测试
内存泄露测试
```

按测试实施的组织：
```
α测试（用户在开发环境下进行的测试）
β测试（用户在实际环境下进行的测试）
第三方测试
```

按是否手工执行划分：
```
手工测试
自动化测试
```

其他分类：
```
冒烟测试
回归测试
```

## Selenium三剑客
Selenium是一个用于Web应用程序自动化测试工具。Selenium测试直接运行
在浏览器中，就像真正的用户在操作。
主要功能包括：
- 测试与浏览器的兼容性——测试应用程序看是否能够很好的工作在不同浏览
器和操作系统之上。
- 测试系统功能——创建回归测试检验软件功能和用户需求。

1. Selenium WebDriver
Selenium WebDriver是客户端API接口，测试人员通过调用这些接口，来访
问浏览器驱动，浏览器驱动再访问浏览器。
![avatar](C:\Users\dwayne\Desktop\极客时间-软件测试\Selenium 自动化测试实战\Pictures\Selenium WebDriver.png)
另外，与浏览器的通信也可以是通过Selenium Server或RemoteWebDriver的
远程通信。RemoteWebDriver与驱动程序和浏览器在同一系统上运行。
![avatar](C:\Users\dwayne\Desktop\极客时间-软件测试\Selenium 自动化测试实战\Pictures\Selenium RemoteDriver.png)
除此之外，还可以使用Selenium Server或Selenium Grid进行分布式测试。
![avatar](C:\Users\dwayne\Desktop\极客时间-软件测试\Selenium 自动化测试实战\Pictures\Selenium Server or Grid.png)

2. Selenium IDE
浏览器插件。

3. Selenium Grid
想通过在多台计算机上进行分布式来扩容，并从一个中心点管理多个环境，从而
轻松地对多种浏览器/OS 组合运行测试，那么可以使用Selenium Grid。

## Selenium开发环境搭建
Python、Selenium、PyCharm、浏览器驱动、

下载对应版本驱动
- 打开Selenium官网：https://www.selenium.dev/
- 选择文档Documentation
- 选择Selenium安装 https://www.selenium.dev/documentation/zh-cn/selenium_installation
- 选择安装WebDriver二进制文件 https://www.selenium.dev/documentation/zh-cn/selenium_installation/installing_webdriver_binaries/
- 选择WebDriver二进制文件 https://www.selenium.dev/documentation/zh-cn/webdriver/driver_requirements/

ChromeDriver官网：
https://chromedriver.chromium.org/downloads

接下后将驱动放置在Python的安装目录下

Selenium 4.0与Selenium 3.0比用法有改变
如3.0 *driver.get_element_by_id('xx')* 改为 4.0 *driver.get_element(By.id, 'xx')*

# 二、Selenium核心技术
## 1. Selenium八大定位

| # | 方法名称 | 描述 |
| -- | -- | -- |
| 1 | find_element(By.ID) | 通过id定位元素 |
| 2 | find_element(By.XPATH) | 通过xpath定位元素 | 
| 3 | find_element(By.LINK_TEXT) | 通过连接文本定位元素 |
| 4 | find_element(By.PARTIAL_LINK_TEXT) | 通过部分链接文本定位 |
| 5 | find_element(By.NAME) | 通过属性标签名称定位 |
| 6 | find_element(By.TAG_NAME) | 通过元素标签名称定位 |
| 7 | find_element(By.CLASS_NAME) | 通过css class定位 |
| 8 | find_element(By.CSS_SELECTOR) | 通过css选择器 |

find_element(By.NAME)方法可能返回多个元素，返回第一个；
find_elements(By.NAME)方法返回一个集合

更简洁的方式去查找页面元素
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from time import sleep

def get_element(driver, *loc):
    e = driver.find_element(*loc)
    return e

if __name__ == '__main__':
    driver = webdriver.Chrome()
    driver.get('https://www.baidu.com')
    sleep(3)
    get_element(driver, By.ID, 'kw').send_keys('selenium')
    get_element(driver, By.ID, 'su').click()
```

## 2. 剖析WebDriver运行原理

Selenium WebDriver和浏览器如何通信
```
对于每一条Selenium脚本，一个http请求会被创建并且发送给浏览器的驱动
浏览器驱动中包含了一个HTTP Server，用来接收这些http请求
HTTP Server接收到请求后根据请求来具体操控对应的浏览器
浏览器执行具体的测试步骤
浏览器将步骤执行结果返回给HTTP Server
HTTP Server又将结果返回给Selenium的脚本，如果是错误的http代码会在控制台看到对应的报错信息
```

WebDriver的协议
```
WebDriver使用的协议是：JSON Wire protocol
通信的数据格式是JSON
```

## 3. 掌握WebDriver核心方法和属性的使用

### Selenium Webdriver属性
| # | 属性 | 属性描述 |
| --- | --- | --- |
| 1 | driver.name | 浏览器名称 | 
| 2 | driver.current_url | 当前url |
| 3 | driver.title | 当前页面标题 |
| 4 | driver.page_source | 当前页面源码 |
| 5 | driver.current_window_handle | 窗口句柄 |
| 6 | driver.window_handles | 当前窗口所有句柄 |

### Selenium WebDriver方法
| # | 方法 | 方法描述 |
| --- | --- | --- |
| 1 | driver.back() | 浏览器后退 |
| 2 | driver.forward() | 浏览器前进 |
| 3 | driver.refresh() | 浏览器刷新 |
| 4 | driver.close() | 关闭当前窗口 |
| 5 | driver.quit() | 推出浏览器 |
| 6 | driver.switch_to.frame() | 切换到frame |
| 7 | driver.switch_to.alert | 切换到alert |
| 8 | driver.switch_to.active_element | 切换到活动元素 |

driver.close() 只关闭当前tab
driver.quit() 关闭浏览器

### WebElemen属性
| # | 属性 | 属性描述 |
| --- | --- | --- |# 
| 1 | id | 标示 |
| 2 | size | 宽高 |
| 3 | rect | 宽高和坐标 |
| 4 | tag_name | 标签名称 |
| 5 | text | 文本内容 |

### WebElement方法
| # | 方法 | 方法描述 |
| --- | --- | --- |
| 1 | send_keys() | 输入内容 |
| 2 | clear() | 清空内容 |
| 3 | click() | 单击 |
| 4 | get_attribute() | 获得属性值 |
| 5 | is_selected() | 是否被选中 |
| 6 | is_enabled() | 是否可用 |
| 7 | is_displayed() | 是否显示 |
| 8 | value_of_css_property() | css属性值 |

### form表单操作步骤

form表单的处理流程：
1. 定位表单元素
2. 输入测试值
3. 判断表单元素属性
4. 获得表单元素属性
5. 提交表单进行验证

## checkbox和radiobutton：

form表单中checkbox是多选框，radiobutton是单选框。

Selenium操作checkbox和radiobutton：
1.checkbox：如果有id属性可以直接通过id定位，如果没有可以通过input标签名称定位，
然后通过type属性过滤。选择或者反选checkbox，使用click()方法。
2.radiobutton：radiobutton有相同的名称，多个值，可以先通过名称获得，然后通过
值判断。或者反选checkbox，使用click()方法。

## Selenium操作下拉列表
下拉列表导包：
```python
from selenium.webdriver.support.select import Select
```

遍历option：
```python
for option in select.options
```

| # | 方法/属性 | 方法/属性描述 |
| --- | --- | --- |
| 1 | select_by_value() | 根据值选择 |
| 2 | select_by_index() | 根据索引选择 |
| 3 | select_by_visible_text | 根据文本选择 |
| 4 | deselect_by_value | 根据值反选 |
| 5 | deselect_by_index | 根据索引反选 |
| 6 | deselect_by_visible_text | 根据文本反选 |
| 7 | deselect_all | 反选所有 |
| 8 | options | 所有选项 |
| 9 | all_selected_optins | 所有选中选项 |
| 10 | first_selected_option | 第一个选择选项 |

## Selenium三种弹窗处理：alert、confirm、prompt

页面上的弹窗有三种：
- alert：用来提示
- confirm：用来确认
- prompt：输入内容

| # | 方法/属性 | 方法/属性描述 |
| --- | --- | --- |
| 1 | accept() | 接受 |
| 2 | dismiss() | 取消 |
| 3 | text | 显示的文本 |
| 4 | send_keys | 输入内容 |

alert切换对象，alet/prompt/confirm都要使用该语句：
```python
alert = driver.switch_to.alert
alert.accept()
```

接受方法：
1. alert
alert.accept()
2. confirm
confirm.accept()
confirm.dismiss()

# 三、Selenium IDE

# 四、项目实战

# 五、使用unittest框架重构项目

# 六、使用pytest框架重构项目

# 七、为项目添加日志

# 八、用DDT思想重构项目

# 九、用POM涉及模式重构项目

# 十、Selenium Grid分布式测试

# 十一、持续集成和交付

