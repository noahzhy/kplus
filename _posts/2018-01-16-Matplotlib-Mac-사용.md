---
layout: post
current: post
cover: False
navigation: True
title: 맥에서 Python Matplotlib Import문제 해결 방법
date: 2018-01-01 22:18:00
tags: develop solution
class: post-template
subclass: 'post tag-develop'
logo: assets/images/slaysd.png
author: slaysd
---

> 필자는 맥에서 virtualenv를 통해 Python을 사용하고 있습니다. 데이터 관련 그래프를 사용하기 위해 `matplotlib`을 설치하고 `import`를 하는 구문에서 다음과 같은 에러가 나타났습니다.

~~~ bash
RuntimeError: Python is not installed as a framework. The Mac OS X backend will not be able to function correctly if Python is not installed as a framework. See the Python documentation for more information on installing Python as a framework on Mac OS X. Please either reinstall Python as a framework, or try one of the other backends. If you are using (Ana)Conda please install python.app and replace the use of 'python' with 'pythonw'. See 'Working with Matplotlib on OSX' in the Matplotlib FAQ for more information.
~~~

맥에서는 `matplotlib`를 사용하는데 있어서 추가적인 조치가 필요하다는 것을 찾았고 해결방법을 찾았습니다.



## 해결방법#1 스크립트 내 명기

Python 스크립트 내에서 사용할 때마다 매번 추가해주는 방법으로 해결이 가능합니다.

~~~python
import matplotlib
matplotlib.use('TkAgg')
~~~

위와 같이 `matplotlib`의 backend를 `TkAgg`로 설정을 하면 오류를 해결할 수 있습니다.

## 해결방법#2 설정파일 명기

저와 같은 경우에는 이 방법을 사용하고 있습니다.

~~~bash
echo "backend: TkAgg" >> ~/.matplotlib/matplotlibrc
~~~
이 방법은 한번 설정만 하면 매 스크립트마다 추가를 해줄 필요가 없기 때문에 가장 편한 방법으로 생각됩니다.

* * *

독자분들도 같은 오류를 뿜어낸다면 위와 같은 방법으로 모두 해결되리라 생각됩니다. 감사합니다.