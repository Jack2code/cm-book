# Pydev

标签（空格分隔）： Python

---

## 准备
1. 安装Python
2. 安装eclipse
3. 安装pydev

## 安装PyDev
1. 配置网络代理（公司内网限制）

2. 从eclipse中直接安装PyDev
进入菜单 `Help` -> `Install New Software...`，
弹出`Install`的对话框中，点`Add`按钮，弹出`Add Repository`对话框：
`Name`：PyDev
`Location`：
点击`OK`，确定添加。
然后点击`Next`,`Next`，选择`I accept the terms of the license agreements`（你懂的），点击`finish`，然后就静候安装完成吧：）

## 配置PyDev解释器
安装好PyDev后， 需要配置Python解释器。
在Eclipse菜单栏中，点击`Windows` -> `Preferences`  
在对话框中，点击`PyDev` -> `Interpreters` - `Python Interpreters`.  点击`New`按钮，
输入`Interpreters Name`（自定义一个名称）和选择`Interpreters Executable`(`python.exe`的路径), 点OK，然后弹出`Selected needed`对话框，默认勾选全部，保持默认即可，点OK。
点OK，保存配置。

## 开始写代码
启动Eclipse,  创建一个新的项目,   File->New->Projects...   选择PyDev->PyDevProject 输入项目名称.

新建 pyDev Package.    就可以写代码了。

