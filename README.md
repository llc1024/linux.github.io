# linux内核工作原理

学号241，姓名刘乐成，原创作品转载请注明出处 + [https://github.com/mengning/linuxkernel/](https://github.com/mengning/linuxkernel/)。

## 实验环境

VMware Workstation 14 Player虚拟机
Ubuntu 64位系统

## 实验目的

完成一个简单的时间片轮转多道程序内核代码，参考代码见[mykernel版本库](https://github.com/mengning/mykernel)。

## 实验步骤

1.在安装QUME前先要更新系统

```markdown
sudo apt-get update
```

2.更新之后安装QUME

```
sudo apt-get install qemu
sudo ln -s /usr/bin/qemu-system-i386 /usr/bin/qemu
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/llc1024/linux.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
