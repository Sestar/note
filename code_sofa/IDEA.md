# IDEA

&nbsp;
## 快捷键

Function|ShortCut|
---|---
eval express | alt + F8  
override method  | Ctrl + O
search class | Ctrl+Shift+Alt+N
search class method | Ctrl + F12
git commit |Ctrl + K
git push | Ctrl + Shift + K
git update |  Ctrl + T
surround code | Ctrl + Alt + T
format code | Ctrl + Shift + L
recent  file | Ctrl + E
code tip | Ctrl + Space
show declare of method | Ctrl + Q
arrive line | Ctrl + G
arrive next error code | F2
arrive previous error code | Shift + F2

</br>

## 插件
</br>

### 添加插件
</br>

#### IDEA客户端直接添加
* Setting -> Plugins -> Browse repositories -> 输入插件名搜索 -> Install -> Restart IDEA
<img src="/note/_v_images/sofa/IDEA/添加插件.png" width="80%" />

</br>

#### 网页直接下载插件

* 官方插件网站: http://plugins.jetbrains.com/idea
* 下载后的插件放在IDEA的根目录的plugins下，重启即可
</br></br>

### 隐藏文件插件(.ignore)

<img src="/note/_v_images/sofa/IDEA/ignore插件使用指南.png" height="600px" />
</br></br>

## 方法注释
</br>

### 方法注释模板

```text
*
 * @Author chengpunan
 * @Description //TODO $end$
 * @Date $time$ $date$
 * @Param $param$
 * @return $return$
 **/
```
</br>

### 注释方法 入参param

```text
  groovyScript("
      def result='';
      def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList();
      for(i = 0; i < params.size(); i++) {
          if (params[i] == '') return result;
          if (i == 0) {
              result += '* @param ' + params[i] + ((i < params.size() - 1) ? ' \\n' : ' ')
          } else {
              result+=' * @param ' + params[i] + ((i < params.size() - 1) ? ' \\n' : ' ')
          }
      };
      return result
  ", methodParameters())
```

```
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {if(params[i] == '') return result; if(i==0) { result += '* @param ' + params[i] + ((i < params.size() - 1) ? ' \\n' : ' ') } else {result+=' * @param ' + params[i] + ((i < params.size() - 1) ? ' \\n' : ' ')}}; return result", methodParameters())
```
</br>

## Application 多次启动

```text
Edit Configurations -> 修改项目 -> 右上角Single instance Only勾选去除
```

<br />

## 随笔

* <span style="color:orange">background task 自动弹出</span>  window->Background Task->Auto show
</br></br>


* <span style="color:orange">IDEA   添加单元测试覆盖率</span>  
  配置启动项: Edit Configurations...  
  
  <img src="/note/_v_images/sofa/IDEA/单元测试覆盖率-1.png" width="70%"/>  
  
  覆盖率是检测运行期间是否所有分支都有执行, 包括异常抛出的分支, 在Repeat可以选择多次, 能让覆盖率报告更加精确, 毕竟执行一次只会执行一个分支。
  
  <img src="/note/_v_images/sofa/IDEA/单元测试覆盖率-2.png" width="50%"/>  
  
  选择不同的范围生成不同测试覆盖报告, 显示覆盖报告点击右键-> Run With Coverage或者Edit Configurations旁边的Run With Coverage
  需要测试覆盖报告文件，在Coverage窗口点击Genergate Coverage Report(左上角红线)即可, Coverage窗口(右下角红线)