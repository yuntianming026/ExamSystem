# ExamSystem
在线题库考试辅助系统
分享一个LiteDB做的简单考试系统辅助工具
阅读目录

1.考试系统辅助需求
2.简单思考后的思路
3.实现过程与代码
4.使用方法
5.资源
    凌晨，被安排在公司值班，因为台风“灿鸿”即将登陆，风力太大，办公楼，车间等重要部分需要关注。所以无聊，那就分享一下，今天给朋友临时做的一个小的考试系统辅助工具吧。其实非常小，需求也很简单，但是可以根据实际需要进行扩充，暂时只实现了一些核心功能。界面丑了点，无所谓，凑合着用吧。



回到目录
1.考试系统辅助需求

    上午10点一个朋友紧急求助，单位要进行在线测评，开卷考试，题库以及答案已经发给他们了，但是太多，好几百道题目，翻资料都来不及。问我能不能做一个软件，能够快速填充答案或者找到题目，节省时间，提高准确率。经过半个小时的QQ沟通，基本明确了大概要做的，由于时间紧急，晚上就要用，尽量搞简单的吧。总结下来，有这么几个需求：

要能快速导入题库，几百道题，手动添加得多长时间，不敢想象，再说了，会写点程序就是要减轻工作量；

要尽量自动化，直接填充答案是不可能了，不是技术上不可行，是时间来不及，在线考试的页面都没见到；

题库类型比较多，有填空题，单选题，多选题以及判断题，要尽量区分，第一时间找到原题和答案；

回到目录
2.简单思考后的思路

    拿到需求，虽然长时间没搞WinForm，但简单的东西还是会的。脑子里第一个飘出的思路就是：

使用Aspose.Words快速导入题库，成熟好用，以前接触过一点点；

由于给定的题库和答案已经有了，暂时就不用区分题目类型了，直接找到题目就找到了答案，然后手动填写就好了

题库数据用什么保存？显然用啥MSSQL，Mysql都是扯淡了，那Sqlite呢？No,No,No，前不久刚刚接触过开源的.NET NoSQL数据库LiteDB，为啥不尝鲜呢？使用文章看这里：

               .NET平台开源项目速览(3)小巧轻量级NoSQL文件数据库LiteDB 

               .NET平台开源项目速览(7)关于NoSQL数据库LiteDB的分页查询解决过程

       4.怎么更快的找到题目呢？难道要手动复制，再粘贴到界面搜索？有点慢，那就先省略一步吧，复制题干部分文字后，直接点界面搜索，然后自动获取剪切板的数据，自动进行搜索，也就是1个题目要2次操作，复制和点击搜索。虽然想到了 监视剪切板，但第一次的思路里面考虑到时间，就没往下想。

       5.如何搜索呢？直接暴力点吧，也就几百道题，就算几千道也应该不成问题，直接题干字段对比是否包括 选择的文字部分，然后搜索，弹出相应的题目；

　　6.考虑到有可能一些文字（复制题干的时候不是全部复制，部分复制就可以了）包含在多个题目中，要至少考虑2个吧，否则就题目复制的时候稍微多一点，减少重复的概率；

　　虽然上面是简单的思路，但实际上最终不完全是这样子，经过实际的修补，不断调整方向，有一些改动：

　　1.Aspose.Words还是不要用了，过程处理有点繁琐，直接搞一个导入题库的界面，文本框和按钮，将题目手动复制进去，几分钟都不需要的事情，何必用Aspose.Words搞来搞去；

　　2.虽然区分类型很简单，开始打算不做，后面还是做了，因为也的确比较简单吧；

回到目录
3.实现过程与代码

    为配合LiteDB数据库的操作，写了个简单的题目实体类，包括简单几个字段，如下所示代码：

1
2
3
4
5
6
7
8
9
10
11
public class Problem
{
    /// <summary>题目编号，可以重复，注意不能使用Id</summary>
    public Int32 ProbId { get; set; }
    /// <summary>整个题目内容，同时包括选项</summary>
    public String ProbText { get; set; }
    /// <summary>答案，一般是题干答案中的东西</summary>
    public String Answer { get; set; }
    /// <summary>题目类型</summary>
    public String TypeName { get; set; }
}
3.1 实现过程之数据导入

    数据导入界面很简单，选择题目类型，然后复制进去，因为已经拿到Word版的题库了，很规则，由于朋友单位内部资料，题库不分享，自己到百度找了一个差不多格式的试卷，如下：



    复制的时候，直接复制题目(要包括编号)即可，前面的 单项选择题，类型就不要复制了，手动选一下吧。看看界面：



    代码很简单，唯一要注意的是，对于选择题来说，如何区分不同题目是个难点，开始没注意这个问题，因为填空题以及判断题都可以通过换行符来确定，换行了就是一道新题目。但是这个选择题就尴尬了，有几次换行，而且还不固定，最后观察到题目的编号是一个可以利用的东西，每一次题目前都是 数字+、号，用这个来区分吧。看看实现代码，这是最终的版本，中间修改的就不说了，涉及到LiteDB的简单操作，可以参考上面的文章或者下面的核心导入按钮的代码：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
//获取题目类型
var typeName = comboBox1.Text.Trim();
using (var db = new LiteDatabase("Prob.db"))
{    
    var col = db.GetCollection<Problem>("Problem");
    //先换行分割
    var prolist = textBox1.Text.Trim().Split(new String[] { "\r\n" }, StringSplitOptions.RemoveEmptyEntries);
    Problem model ;
    Int32 index = 0;
    //题目列表，多行的每次读最后一个
    List<Problem> modelList = new List<Problem>();
    //根据每行的第一个字符和、号进行区分，如果不是新题目，就作为选项添加到上一个题目中去
    while (index <prolist.Length )
    {
        var item = prolist[index];
        if (String.IsNullOrEmpty(item)) continue;
        var titles = item.Trim().Split('、');
        if (titles.Length > 0)
        {
            int number;
            if(Int32.TryParse(titles[0],out number))
            {   //是数字，添加
                model = new Problem();  //如果分割第1个是数字，则说明是1个新的题目
                model.ProbId = number;
                model.ProbText = item;
                model.TypeName = typeName;
                //确定答案，答案在（）里面，把括号里面是字符串分解并组合
                var ans = item.Split('（', '）');
                if (ans.Length < 2) model.Answer = string.Empty;
                if (ans.Length > 1) model.Answer = ans[1];
                if (ans.Length > 3) model.Answer = ans[1] + "\r\n" + ans[3];
                modelList.Add(model);
            }
            else
            {
                //不是数字，就添加到前一个实体中去，并更新题目内容
                modelList.Last().ProbText += ("\r\n" + item);
            }
        }
        else
        {
            //不是数字，就添加到前一个实体中去，并更新题目内容
            modelList.Last().ProbText += ("\r\n" + item);
        }
        index++;
    }
    col.InsertBulk(modelList);
}
    当然每个人遇到的题库格式不一样，自己在这里修改呗。。。我也没时间做个通用的。。反正是给朋友帮忙的东西。顺便分享给大家解决问题的思路。

3.2 实现过程之搜索主界面

    搜索界面也是很简单，第一次实现的时候没有考虑监视ctrl+c，只是想点击搜索，自动获取剪切板内容进行搜索。再就是要注意多条记录的问题，只显示2条吧，如果找不到相同的，就复制多一点题目部分。每一个题目显示题目类型，答案，如果为空就显示手动选择档案。界面如下：



    核心代码如下：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
//每次搜索前都要清空其他控件
txtDisp1.Text = String.Empty;
txtDisp2.Text = String.Empty;
lblAnswer1.Text = String.Empty;
lblAnswer2.Text = String.Empty;
try
{
    //获取剪切板数据
    IDataObject iData = Clipboard.GetDataObject();
    if (iData.GetDataPresent(DataFormats.Text)) txtKeyValue.Text = ((String)iData.GetData(DataFormats.Text)).Trim();
    //查找满足关键词的题目
    var models = list.Where(n => n.ProbText.Contains(txtKeyValue.Text)).ToList();
    //最多只显示2个包括选择关键词的题目
    if (models.Count == 1)
    {
        lblTypeName1.Text = models[0].TypeName;
        lblAnswer1.Text = String.IsNullOrEmpty(models[0].Answer) ? "题目寻找答案" : models[0].Answer;
        txtDisp1.Text = models[0].ProbText;
    }
    if (models.Count >= 2)
    {
        lblTypeName1.Text = models[0].TypeName;
        lblAnswer1.Text = String.IsNullOrEmpty(models[0].Answer) ? "题目寻找答案" : models[0].Answer;
        txtDisp1.Text = models[0].ProbText;
 
        lblTypeName2.Text = models[1].TypeName;
        lblAnswer2.Text = String.IsNullOrEmpty(models[1].Answer) ? "题目寻找答案" : models[1].Answer;
        txtDisp2.Text = models[1].ProbText;
    }
}
catch(Exception err)
{
    MessageBox.Show("出现错误,重启后再试！");
}
3.3 监视复制Ctrl+C

    因为还有时间，所以就想尽量改善点，然后有去百度如何 监视剪切板，貌似要用到win api，学艺不精，搞不来，看到一篇文章监视ctrl+c，其实也是个不错的方案，一步步来吗，哪位童鞋有C# 直接监视剪切板的 代码，希望分享一下。

    我用到的代码来源于这里：http://www.cnblogs.com/over140/archive/2007/11/05/934452.html

    自己稍微修改下就OK了，不详细解释了。

3.4 塞翁失马-手贱的教训

    这个完整的东西前后共用了大约4个小时整的时间。坑爹的是中午写的第一版代码，让我手动删除目录给删错了。。。后来改进的时候，无奈又重新写了一下，其实思路都有，代码比较简单。

以前习惯频繁更新SVN的，这次失手，算教训吧，幸好不是啥重要代码。另外，塞翁失马焉知非福，重新写也有好处，不用看改得乱七八糟的代码，一次到位，代码思路也清晰多了，初版本代码这里有的是测试，临时使用。。。等等。算是一种安慰吧。

回到目录
4.使用方法

    使用方法这再次简单介绍一下，先手动复制题库到导入界面，进行题库导入；然后启动进行搜索，可以将在线考试的页面和该软件同时打开，分屏放置，通过Ctrl+C或者手动复制，搜索也可以进行，查询到题目后，对照答案填写，然后循环使用。如果没有答案，那就看原题，原题一般是有答案的，除非格式有变动，无法转换。

回到目录
5.资源

    代码托管到Github吧，我不会告诉你地址在这里的：https://github.com/asxinyu/ExamSystem/

    不知不觉，抽了几根烟，已经过去1个多小时了，“灿鸿”已经越来越近，办公楼窗户以及墙体渗水一塌糊涂，去抢险去了，抢险完了，要去睡觉，希望醒来的时候，看到大伙的点赞呢。


如果您觉得阅读本文对您有帮助，请点一下“推荐”按钮，您的“推荐”将是我最大的写作动力！欢迎各位转载，但是未经作者本人同意，转载文章之后必须在文章页面明显位置给出作者和原文连接，否则保留追究法律责任的权利。
.NET数据挖掘与机器学习，作者博客: http://www.cnblogs.com/asxinyu

E-mail:1287263703@qq.com

分类: .NET开源项目, C#.NET开发, 工具资源, 开源技术, 源代码
标签: LiteDB数据库, .NET考题系统, C#考试辅助, C#辅助, 考题导入, 寻找考题答案
好文要顶 已关注 收藏该文    
数据之巅
关注 - 43
粉丝 - 2520
荣誉：推荐博客
我在关注他 取消关注
26 3
(请您对文章做出评价)
« 上一篇：.NET平台开源项目速览(9)软件序列号生成组件SoftwareProtector介绍与使用
» 下一篇：.NET平台开源项目速览(10)FluentValidation验证组件深入使用(二)
posted @ 2015-07-11 03:12 数据之巅 阅读(3083) 评论(11) 编辑 收藏
