# 增加语言方案

## 【问题描述】

客户需求,要求日文可以在平假字和汉字之间切换.

 

## 【分析过程】

刚开始拿到问题的是,查找了日文字库的支持格式,发现日本字库就是包括了平假字和汉字,是这两个的一个集合.不存在两个

字库的方式.

 

## 【解决方法】

日文包括了平假字和汉字的集合,那么如果我们要切换平假字和汉字,我们就需要新建一个语言,日文作为平假字的设置,新建

的语言作为汉字的设置方式.

添加一种新的语言

 

#### lily方案：

### 1 确认语言类型

language\_code  与 country code   比如 ak-GH     language code 就是 ak ， country code就是GH 

资源文件夹可以定义为values-ak-rGH  如果没有国家码冲突，可以直接定义 values-ak

&gt; 1. 我们需要去下面网站查找 language code 和 contry code:
&gt;
&gt; language code: https://zh.wikipedia.org/wiki/ISO\_639­1
&gt;
&gt; country code : https://zh.wikipedia.org/wiki/ISO\_3166­1
&gt;
&gt; 如果我们添加的语言在里面,就直接使用,如果不在里面,那我们需要确保我们添加的缩写不能与上面的ISO\_639­1重复.
&gt;
&gt; 所以我们以hg\_JP 作为新添加的一种语言,hg是 language cod, JP是conrty code
&gt;
&gt;  

由于手表项目 语言客户指定了平假名的 语言类型 ak ，为了适配客户apk，在手表项目中没有新建一个语言，而是在现有ak的语言基础上做修改。同样需要到icu中更改icu数据。

 

### 2 添加ICU资源

由于Hiragana/Kanji是给上层语言切换做区分,ICU的资源我们主要是拷贝ja 里面的资源，再进行客制化的修改。

把对应的txt文件放到（external\icu\icu4c\source\data）目录下coll、curr、lang、locales、region，zone这些子文件夹

中。

 

主要是检查上面目录里面的ak.txt 文件， 将ja.txt 拷贝至ak.txt文件

 

主要差异话的，locales 里面的txt文件：

#### 文字差异 午前 午後

```d
1. --- a/icu4c/source/data/locales/ja.txt
2. &#43;&#43;&#43; b/icu4c/source/data/locales/ja.txt
3. @@ -1258,16 &#43;1258,16 @@
4.          }
5.          gregorian{
6.              AmPmMarkers{
7. -                &#34;午前&#34;,
8. -                &#34;午後&#34;,
9. &#43;                &#34;ごぜん&#34;,
10. &#43;                &#34;ごご&#34;,
11.              }
12.              AmPmMarkersAbbr{
13. -                &#34;午前&#34;,
14. -                &#34;午後&#34;,
15. &#43;                &#34;ごぜん&#34;,
16. &#43;                &#34;ごご&#34;,
17.              }
18.              AmPmMarkersNarrow{
19. -                &#34;午前&#34;,
20. -                &#34;午後&#34;,
21. &#43;                &#34;ごぜん&#34;,
22. &#43;                &#34;ごご&#34;,
23.              }
24.              DateTimePatterns{
25.                  &#34;H時mm分ss秒 zzzz&#34;,
26. @@ -1345,30 &#43;1345,30 @@
27.                  format{
28.                      abbreviated{
29.                          &#34;日&#34;,
30. -                        &#34;月&#34;,
31. -                        &#34;火&#34;,
32. -                        &#34;水&#34;,
33. -                        &#34;木&#34;,
34. -                        &#34;金&#34;,
35. -                        &#34;土&#34;,
36. &#43;                        &#34;にち&#34;,
37. &#43;                        &#34;げつ&#34;,
38. &#43;                        &#34;か&#34;,
39. &#43;                        &#34;すい&#34;,
40. &#43;                        &#34;もく&#34;,
41. &#43;                        &#34;きん&#34;,
42. &#43;                        &#34;ど&#34;,
43.                      }
```

 

#### 2.1 更新分词规则

![image.png](https://picgo.myjojo.fun:666/i/2024/03/12/65efc4cc53811.png)\
客户报出问题，不同语言下，同样字符串，换行不一样。\
经过分析。TextView 里面的换行规则是在StaticLayout里面：

 

```java
LineBreaker.Result res = lineBreaker.computeLineBreaks(measuredPara.getMeasuredText(), constraints, mLineCount);
```

```java
    /**
     * Break paragraph into lines.
     *
     * The result is filled to out param.
     *
     * @param measuredPara a result of the text measurement
     * @param constraints for a single paragraph
     * @param lineNumber a line number of this paragraph
     */
    public @NonNull Result computeLineBreaks(
            @NonNull MeasuredText measuredPara,
            @NonNull ParagraphConstraints constraints,
            @IntRange(from = 0) int lineNumber) {

        return new Result(nComputeLineBreaks(
                mNativePtr,

                // Inputs
                measuredPara.getChars(),
                measuredPara.getNativePtr(),
                measuredPara.getChars().length,
                constraints.mFirstWidth,
                constraints.mFirstWidthLineCount,
                constraints.mWidth,
                constraints.mVariableTabStops,
                constraints.mDefaultTabStop,
                lineNumber));
    }
```

   nComputeLineBreaks 方法是一个Native的方法，用于计算什么时候开始换行

![image.png](https://picgo.myjojo.fun:666/i/2024/03/12/65efc5dcaad6f.png)

发现换行要根据分词规则去换行，那么日语ja有特定的分词规则，而额外添加的ak没有，使用的就是基础的分词规则。\
如果要保持一致，那么就需要将ja的brkitr 资源拷贝一份给ak

&gt; \[!NOTE] brkitr 说明\
&gt; ICU（International Components for Unicode）的 `brkitr` 库是一个提供文本边界分析的库，`brkitr` 是“boundary breaker iterator”（边界分割迭代器）的缩写。它用来确定文本中的边界位置，如单词边界、句子边界、行边界和字符边界。这对于文本处理非常关键，尤其是在需要对文本进行分词、换行、选择或光标移动等操作时。

以下是 `brkitr` 库一些常见的使用场景：

&gt; 1. **单词边界**：用于文本编辑或处理时确定单词的开始和结束，这在文本选择或者双击单词时特别有用。
&gt;
&gt; 2. **句子边界**：用于文本处理时确定句子的开始和结束，例如，在处理自动大写新句子的首字母或文本摘要时。
&gt;
&gt; 3. **行边界**：用于文本布局时确定在哪里进行换行，这在UI组件如TextView的文本折行处理中尤其重要。
&gt;
&gt; 4. **字符边界**：用于确定可分割的字符边界，比如在处理复合字符（如带变音符的字母）时正确地光标定位或者文本选择。

`brkitr` 库实现了国际化的文本边界规则，支持多种语言和文本规则。它利用Unicode文本分割算法和语言特定的规则来确定这些边界，从而提供准确的文本处理能力。在全球化的应用开发中，正确地识别和处理不同语言的文本边界是非常重要的。

### 3. 编译ICU资源

&gt; lili项目
&gt;
&gt; 进入到\$AOSP/external/icu/icu4c/source/目录下的
&gt;
&gt; 在该目录下执行 ./runConfigureICU Linux命令生成MAKE文件
&gt;
&gt; 执行make INCLUDE\_UNI\_CORE\_DATA=1
&gt;
&gt; 1. 将第一步生成的external/icu4c/icubuild/data/out/tmp/icudtxxl.dat复制到external/icu4c/stubdata下并改名为
&gt;
&gt; icudtxxl­all.dat覆盖原来的同名文件。
&gt;
&gt; cpexternal/icu/icu4c/source/data/out/tmp/icudt58l.dat \$AOSP/external/icu/icu4c/source/stubdata 

该方案在A11 上编译失效，烧入手机后 ，载入icu会出错 开不了机器。

编译方案：

首先在工程根目录，进行source ,lunch ，设置环境变量。

然后在  aosp/external/icu/tools 目录下

执行下面脚本：updateicudata.py

会自动config 和编译，然后自动把结果copy 到 stubdata目录。

 

 

 

 

 

### 4.添加语言支持

\# [sdm429w\_law.mk](http://review4-sh.sim.com:8080/c/Android-HighPlat-Repositories/platform/vendor/qcom/sdm429w_law/&#43;/46394/1/sdm429w_law.mk) Definedthelocales

PRODUCT\_LOCALES:=en\_US  ja\_JP   ak\_GK

 

 

 

 

### 5.翻译字符串

在frameworks/base/core/res/res/下新增加一个values­-ak的文件夹,新建一个strings.xml文件，把frameworks翻

译内容放在这个文件内:

对每个app做翻译，在每个app对应的res目录下面建立values­-ak文件夹，并将翻译好的strings.xml放在里面；

 

 

### 6. 添加设置菜单

 

设置语言主要就是通过  LocalePicker.updateLocale(hiragana); 来实现

 

1. import java.util.Locale;

2. import com.android.internal.app.LocalePicker;

3.     static final String hiragana\_language =&#34;ak&#34;;

4.     static final String hiragana\_country =&#34;GH&#34;;

5.     static final String kanji\_language = &#34;ja&#34;;

6.     static final String kanji\_country = &#34;JP&#34;;

7.     static final String english\_language = &#34;en&#34;;

8.     static final String english\_country = &#34;US&#34;;

9.     @Override

10.     protected void onCreate(Bundle savedInstanceState) {

11.         super.onCreate(savedInstanceState);

12.         setContentView(R.layout.activity\_language\_setting);

13.         mRadioGroup = findViewById(R.id.rg\_show);

14.         mRbHiragana = findViewById(R.id.rb\_hiragana);

15.         mRbKanji    = findViewById(R.id.rb\_kanji);

16.         mRbEnglish  = findViewById(R.id.rb\_en\_us);

17.         updateRadioMenuChecked();

18.         updateEnglisMenu();

19.         mRadioGroup.setOnCheckedChangeListener(new RadioGroup.OnCheckedChangeListener() {

20.             @Override

21.             public void onCheckedChanged(RadioGroup group, int checkedId) {

22.                 if(checkedId == R.id.rb\_hiragana){

23.                     Locale hiragana = new Locale(hiragana\_language,hiragana\_country);

24.                     LocalePicker.updateLocale(hiragana);

25.                 }else if(checkedId == R.id.rb\_kanji) {

26.                     Locale kanji = new Locale(kanji\_language,kanji\_country);

27.                     LocalePicker.updateLocale(kanji);

28.                 }else if(checkedId == R.id.rb\_en\_us) {

29.                     Locale english\_us = new Locale(english\_language,english\_country);

30.                     LocalePicker.updateLocale(english\_us);

31.                 }

32.             }

33.         });

34.     }

### 附录：

##### 年月日翻译：

```d
diff --git a/icu4c/source/data/locales/ak.txt b/icu4c/source/data/locales/ak.txt
index 2591869..2e4ceaf 100644
--- a/icu4c/source/data/locales/ak.txt
&#43;&#43;&#43; b/icu4c/source/data/locales/ak.txt
@@ -243,8 &#43;243,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;GGGGy年M月d日EEEE&#34;,
-                &#34;GGGGy年M月d日&#34;,
&#43;                &#34;GGGGyねんMがつdにちEEEE&#34;,
&#43;                &#34;GGGGyねんMがつdにち&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;{1} {0}&#34;,
@@ -254,42 &#43;254,42 @@
                 &#34;{1} {0}&#34;,
             }
             availableFormats{
-                EEEEd{&#34;d日EEEE&#34;}
-                Ed{&#34;d日(E)&#34;}
-                Gy{&#34;GGGGy年&#34;}
-                GyMMM{&#34;GGGGy年M月&#34;}
-                GyMMMEEEEd{&#34;GGGGy年M月d日EEEE&#34;}
-                GyMMMEd{&#34;GGGGy年M月d日(E)&#34;}
-                GyMMMd{&#34;GGGGy年M月d日&#34;}
-                GyMd{&#34;GGGGy年M月d日&#34;}
&#43;                EEEEd{&#34;dにちEEEE&#34;}
&#43;                Ed{&#34;dにち(E)&#34;}
&#43;                Gy{&#34;GGGGyねん&#34;}
&#43;                GyMMM{&#34;GGGGyねんMがつ&#34;}
&#43;                GyMMMEEEEd{&#34;GGGGyねんMがつdにちEEEE&#34;}
&#43;                GyMMMEd{&#34;GGGGyねんMがつdにち(E)&#34;}
&#43;                GyMMMd{&#34;GGGGyねんMがつdにち&#34;}
&#43;                GyMd{&#34;GGGGyねんMがつdにち&#34;}
                 H{&#34;H時&#34;}
                 Hm{&#34;H:mm&#34;}
                 Hms{&#34;H:mm:ss&#34;}
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEEEEd{&#34;M/dEEEE&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEEEEd{&#34;M月d日EEEE&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEEEEd{&#34;MがつdにちEEEE&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
&#43;                d{&#34;dにち&#34;}
                 h{&#34;aK時&#34;}
                 hm{&#34;aK:mm&#34;}
                 hms{&#34;aK:mm:ss&#34;}
                 ms{&#34;mm:ss&#34;}
-                y{&#34;GGGGy年&#34;}
-                yyyy{&#34;GGGGy年&#34;}
-                yyyyM{&#34;GGGGy年M月&#34;}
-                yyyyMEEEEd{&#34;GGGGy年M/dEEEE&#34;}
-                yyyyMEd{&#34;GGGGy年M/d(E)&#34;}
&#43;                y{&#34;GGGGyねん&#34;}
&#43;                yyyy{&#34;GGGGyねん&#34;}
&#43;                yyyyM{&#34;GGGGyねんMがつ&#34;}
&#43;                yyyyMEEEEd{&#34;GGGGyねんM/dEEEE&#34;}
&#43;                yyyyMEd{&#34;GGGGyねんM/d(E)&#34;}
                 yyyyMM{&#34;Gy/MM&#34;}
-                yyyyMMM{&#34;GGGGy年M月&#34;}
-                yyyyMMMEEEEd{&#34;GGGGy年M月d日EEEE&#34;}
-                yyyyMMMEd{&#34;GGGGy年M月d日(E)&#34;}
-                yyyyMMMM{&#34;GGGGy年M月&#34;}
-                yyyyMMMd{&#34;GGGGy年M月d日&#34;}
&#43;                yyyyMMM{&#34;GGGGyねんMがつ&#34;}
&#43;                yyyyMMMEEEEd{&#34;GGGGyねんMがつdにちEEEE&#34;}
&#43;                yyyyMMMEd{&#34;GGGGyねんMがつdにち(E)&#34;}
&#43;                yyyyMMMM{&#34;GGGGyねんMがつ&#34;}
&#43;                yyyyMMMd{&#34;GGGGyねんMがつdにち&#34;}
                 yyyyMd{&#34;Gy/M/d&#34;}
             }
             eras{
@@ -311,15 &#43;311,15 @@
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
                 {
-                    &#34;U年MMMd日EEEE&#34;,
&#43;                    &#34;UねんMMMdにちEEEE&#34;,
                     &#34;hanidec&#34;,
                 }
                 {
-                    &#34;U年MMMd日&#34;,
&#43;                    &#34;UねんMMMdにち&#34;,
                     &#34;hanidec&#34;,
                 }
                 {
-                    &#34;U年MMMd日&#34;,
&#43;                    &#34;UねんMMMdにち&#34;,
                     &#34;hanidec&#34;,
                 }
                 &#34;U-M-d&#34;,
@@ -336,13 &#43;336,13 @@
                 E{&#34;ccc&#34;}
                 EBhm{&#34;BK:mm (E)&#34;}
                 EBhms{&#34;BK:mm:ss (E)&#34;}
-                EEEEd{&#34;d日EEEE&#34;}
-                Ed{&#34;d日(E)&#34;}
-                Gy{&#34;U年&#34;}
-                GyMMM{&#34;U年MMM&#34;}
-                GyMMMEEEEd{&#34;U年MMMd日EEEE&#34;}
-                GyMMMEd{&#34;U年MMMd日(E)&#34;}
-                GyMMMd{&#34;U年MMMd日&#34;}
&#43;                EEEEd{&#34;dにちEEEE&#34;}
&#43;                Ed{&#34;dにち(E)&#34;}
&#43;                Gy{&#34;Uねん&#34;}
&#43;                GyMMM{&#34;UねんMMM&#34;}
&#43;                GyMMMEEEEd{&#34;UねんMMMdにちEEEE&#34;}
&#43;                GyMMMEd{&#34;UねんMMMdにち(E)&#34;}
&#43;                GyMMMd{&#34;UねんMMMdにち&#34;}
                 H{&#34;H時&#34;}
                 Hm{&#34;H:mm&#34;}
                 Hms{&#34;H:mm:ss&#34;}
@@ -350,34 &#43;350,34 @@
                 MEEEEd{&#34;M/dEEEE&#34;}
                 MEd{&#34;M/d(E)&#34;}
                 MMM{&#34;LLL&#34;}
-                MMMEEEEd{&#34;MMMd日EEEE&#34;}
-                MMMEd{&#34;MMMd日(E)&#34;}
-                MMMMd{&#34;MMMMd日&#34;}
-                MMMd{&#34;MMMd日&#34;}
&#43;                MMMEEEEd{&#34;MMMdにちEEEE&#34;}
&#43;                MMMEd{&#34;MMMdにち(E)&#34;}
&#43;                MMMMd{&#34;MMMMdにち&#34;}
&#43;                MMMd{&#34;MMMdにち&#34;}
                 Md{&#34;M/d&#34;}
-                UM{&#34;U年M月&#34;}
-                UMMM{&#34;U年MMM&#34;}
-                UMMMd{&#34;U年MMMd日&#34;}
-                UMd{&#34;U年M月d日&#34;}
-                d{&#34;d日&#34;}
&#43;                UM{&#34;UねんMがつ&#34;}
&#43;                UMMM{&#34;UねんMMM&#34;}
&#43;                UMMMd{&#34;UねんMMMdにち&#34;}
&#43;                UMd{&#34;UねんMがつdにち&#34;}
&#43;                d{&#34;dにち&#34;}
                 h{&#34;aK時&#34;}
                 hm{&#34;aK:mm&#34;}
                 hms{&#34;aK:mm:ss&#34;}
                 ms{&#34;mm:ss&#34;}
-                y{&#34;U年&#34;}
-                yMd{&#34;U年M月d日&#34;}
-                yyyy{&#34;U年&#34;}
-                yyyyM{&#34;U年M月&#34;}
-                yyyyMEEEEd{&#34;U年M月d日EEEE&#34;}
-                yyyyMEd{&#34;U年M月d日(E)&#34;}
-                yyyyMMM{&#34;U年MMM&#34;}
-                yyyyMMMEEEEd{&#34;U年MMMd日EEEE&#34;}
-                yyyyMMMEd{&#34;U年MMMd日(E)&#34;}
-                yyyyMMMM{&#34;U年MMMM&#34;}
-                yyyyMMMd{&#34;U年MMMd日&#34;}
-                yyyyMd{&#34;U年M月d日&#34;}
-                yyyyQQQ{&#34;U年QQQ&#34;}
-                yyyyQQQQ{&#34;U年QQQQ&#34;}
&#43;                y{&#34;Uねん&#34;}
&#43;                yMd{&#34;UねんMがつdにち&#34;}
&#43;                yyyy{&#34;Uねん&#34;}
&#43;                yyyyM{&#34;UねんMがつ&#34;}
&#43;                yyyyMEEEEd{&#34;UねんMがつdにちEEEE&#34;}
&#43;                yyyyMEd{&#34;UねんMがつdにち(E)&#34;}
&#43;                yyyyMMM{&#34;UねんMMM&#34;}
&#43;                yyyyMMMEEEEd{&#34;UねんMMMdにちEEEE&#34;}
&#43;                yyyyMMMEd{&#34;UねんMMMdにち(E)&#34;}
&#43;                yyyyMMMM{&#34;UねんMMMM&#34;}
&#43;                yyyyMMMd{&#34;UねんMMMdにち&#34;}
&#43;                yyyyMd{&#34;UねんMがつdにち&#34;}
&#43;                yyyyQQQ{&#34;UねんQQQ&#34;}
&#43;                yyyyQQQQ{&#34;UねんQQQQ&#34;}
             }
             cyclicNameSets{
                 dayParts{
@@ -529,7 &#43;529,7 @@
                     H{&#34;H時～H時(v)&#34;}
                 }
                 M{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MEd{
                     M{&#34;MM/dd(E)～MM/dd(E)&#34;}
@@ -539,22 &#43;539,22 @@
                     M{&#34;MMM～MMM&#34;}
                 }
                 MMMEd{
-                    M{&#34;MMMd日(E)～MMMd日(E)&#34;}
-                    d{&#34;MMMd日(E)～d日(E)&#34;}
&#43;                    M{&#34;MMMdにち(E)～MMMdにち(E)&#34;}
&#43;                    d{&#34;MMMdにち(E)～dにち(E)&#34;}
                 }
                 MMMM{
                     M{&#34;MMMM～MMMM&#34;}
                 }
                 MMMd{
-                    M{&#34;MMMd日～MMMd日&#34;}
-                    d{&#34;MMMd日～d日&#34;}
&#43;                    M{&#34;MMMdにち～MMMdにち&#34;}
&#43;                    d{&#34;MMMdにち～dにち&#34;}
                 }
                 Md{
                     M{&#34;MM/dd～MM/dd&#34;}
                     d{&#34;MM/dd～MM/dd&#34;}
                 }
                 d{
-                    d{&#34;d日～d日&#34;}
&#43;                    d{&#34;dにち～dにち&#34;}
                 }
                 fallback{&#34;{0}～{1}&#34;}
                 h{
@@ -576,7 &#43;576,7 @@
                     h{&#34;aK時～K時(v)&#34;}
                 }
                 y{
-                    y{&#34;U年～U年&#34;}
&#43;                    y{&#34;Uねん～Uねん&#34;}
                 }
                 yM{
                     M{&#34;U/MM～U/MM&#34;}
@@ -588,22 &#43;588,22 @@
                     y{&#34;U/MM/dd(E)～U/MM/dd(E)&#34;}
                 }
                 yMMM{
-                    M{&#34;U年MMM～MMM&#34;}
-                    y{&#34;U年MMM～U年MMM&#34;}
&#43;                    M{&#34;UねんMMM～MMM&#34;}
&#43;                    y{&#34;UねんMMM～UねんMMM&#34;}
                 }
                 yMMMEd{
-                    M{&#34;U年MMMd日(E)～MMMd日(E)&#34;}
-                    d{&#34;U年MMMd日(E)～d日(E)&#34;}
-                    y{&#34;U年MMMd日(E)～U年MMMd日(E)&#34;}
&#43;                    M{&#34;UねんMMMdにち(E)～MMMdにち(E)&#34;}
&#43;                    d{&#34;UねんMMMdにち(E)～dにち(E)&#34;}
&#43;                    y{&#34;UねんMMMdにち(E)～UねんMMMdにち(E)&#34;}
                 }
                 yMMMM{
-                    M{&#34;U年MMM～MMM&#34;}
-                    y{&#34;U年MMM～U年MMM&#34;}
&#43;                    M{&#34;UねんMMM～MMM&#34;}
&#43;                    y{&#34;UねんMMM～UねんMMM&#34;}
                 }
                 yMMMd{
-                    M{&#34;U年MMMd日～MMMd日&#34;}
-                    d{&#34;U年MMMd日～d日&#34;}
-                    y{&#34;U年MMMd日～U年MMMd日&#34;}
&#43;                    M{&#34;UねんMMMdにち～MMMdにち&#34;}
&#43;                    d{&#34;UねんMMMdにち～dにち&#34;}
&#43;                    y{&#34;UねんMMMdにち～UねんMMMdにち&#34;}
                 }
                 yMd{
                     M{&#34;U/MM/dd～U/MM/dd&#34;}
@@ -837,9 &#43;837,9 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;U年MMMd日EEEE&#34;,
-                &#34;U年MMMd日&#34;,
-                &#34;U年MMMd日&#34;,
&#43;                &#34;UねんMMMdにちEEEE&#34;,
&#43;                &#34;UねんMMMdにち&#34;,
&#43;                &#34;UねんMMMdにち&#34;,
                 &#34;U-M-d&#34;,
                 &#34;{1} {0}&#34;,
                 &#34;{1} {0}&#34;,
@@ -1160,8 &#43;1160,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;Gy年M月d日(EEEE)&#34;,
-                &#34;Gy年M月d日&#34;,
&#43;                &#34;GyねんMがつdにち(EEEE)&#34;,
&#43;                &#34;GyねんMがつdにち&#34;,
                 &#34;GGGGGy/MM/dd&#34;,
                 &#34;GGGGGy/M/d&#34;,
                 &#34;{1} {0}&#34;,
@@ -1177,47 &#43;1177,47 @@
                 E{&#34;ccc&#34;}
                 EBhm{&#34;BK:mm (E)&#34;}
                 EBhms{&#34;BK:mm:ss (E)&#34;}
-                EEEEd{&#34;d日(EEEE)&#34;}
&#43;                EEEEd{&#34;dにち(EEEE)&#34;}
                 EHm{&#34;H:mm (E)&#34;}
                 EHms{&#34;H:mm:ss (E)&#34;}
-                Ed{&#34;d日(E)&#34;}
&#43;                Ed{&#34;dにち(E)&#34;}
                 Ehm{&#34;aK:mm (E)&#34;}
                 Ehms{&#34;aK:mm:ss (E)&#34;}
-                Gy{&#34;Gy年&#34;}
-                GyMMM{&#34;Gy年M月&#34;}
-                GyMMMEEEEd{&#34;Gy年M月d日(EEEE)&#34;}
-                GyMMMEd{&#34;Gy年M月d日(E)&#34;}
-                GyMMMd{&#34;Gy年M月d日&#34;}
&#43;                Gy{&#34;Gyねん&#34;}
&#43;                GyMMM{&#34;GyねんMがつ&#34;}
&#43;                GyMMMEEEEd{&#34;GyねんMがつdにち(EEEE)&#34;}
&#43;                GyMMMEd{&#34;GyねんMがつdにち(E)&#34;}
&#43;                GyMMMd{&#34;GyねんMがつdにち&#34;}
                 H{&#34;H時&#34;}
                 Hm{&#34;H:mm&#34;}
                 Hms{&#34;H:mm:ss&#34;}
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEEEEd{&#34;M/d(EEEE)&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEEEEd{&#34;M月d日(EEEE)&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEEEEd{&#34;Mがつdにち(EEEE)&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
&#43;                d{&#34;dにち&#34;}
                 h{&#34;aK時&#34;}
                 hm{&#34;aK:mm&#34;}
                 hms{&#34;aK:mm:ss&#34;}
                 ms{&#34;mm:ss&#34;}
-                y{&#34;Gy年&#34;}
-                yyyy{&#34;Gy年&#34;}
&#43;                y{&#34;Gyねん&#34;}
&#43;                yyyy{&#34;Gyねん&#34;}
                 yyyyM{&#34;GGGGGy/M&#34;}
                 yyyyMEEEEd{&#34;GGGGGy/M/d(EEEE)&#34;}
                 yyyyMEd{&#34;GGGGGy/M/d(E)&#34;}
-                yyyyMMM{&#34;Gy年M月&#34;}
-                yyyyMMMEEEEd{&#34;Gy年M月d日(EEEE)&#34;}
-                yyyyMMMEd{&#34;Gy年M月d日(E)&#34;}
-                yyyyMMMM{&#34;Gy年M月&#34;}
-                yyyyMMMd{&#34;Gy年M月d日&#34;}
&#43;                yyyyMMM{&#34;GyねんMがつ&#34;}
&#43;                yyyyMMMEEEEd{&#34;GyねんMがつdにち(EEEE)&#34;}
&#43;                yyyyMMMEd{&#34;GyねんMがつdにち(E)&#34;}
&#43;                yyyyMMMM{&#34;GyねんMがつ&#34;}
&#43;                yyyyMMMd{&#34;GyねんMがつdにち&#34;}
                 yyyyMd{&#34;GGGGGy/M/d&#34;}
                 yyyyQQQ{&#34;Gy/QQQ&#34;}
-                yyyyQQQQ{&#34;Gy年QQQQ&#34;}
&#43;                yyyyQQQQ{&#34;GyねんQQQQ&#34;}
             }
             intervalFormats{
                 Bh{
@@ -1247,8 &#43;1247,8 @@
                     y{&#34;GGGGGy/MM/dd～y/MM/dd&#34;}
                 }
                 Gy{
-                    G{&#34;Gy年～Gy年&#34;}
-                    y{&#34;Gy年～y年&#34;}
&#43;                    G{&#34;Gyねん～Gyねん&#34;}
&#43;                    y{&#34;Gyねん～yねん&#34;}
                 }
                 GyM{
                     G{&#34;GGGGGy/MM～GGGGGy/MM&#34;}
@@ -1262,21 &#43;1262,21 @@
                     y{&#34;GGGGGy/MM/dd(E)～y/MM/dd(E)&#34;}
                 }
                 GyMMM{
-                    G{&#34;Gy年M月～Gy年M月&#34;}
-                    M{&#34;Gy年M月～M月&#34;}
-                    y{&#34;Gy年M月～y年M月&#34;}
&#43;                    G{&#34;GyねんMがつ～GyねんMがつ&#34;}
&#43;                    M{&#34;GyねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;GyねんMがつ～yねんMがつ&#34;}
                 }
                 GyMMMEd{
-                    G{&#34;Gy年M月d日(E)～Gy年M月d日(E)&#34;}
-                    M{&#34;Gy年M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;Gy年M月d日(E)～d日(E)&#34;}
-                    y{&#34;Gy年M月d日(E)～y年M月d日(E)&#34;}
&#43;                    G{&#34;GyねんMがつdにち(E)～GyねんMがつdにち(E)&#34;}
&#43;                    M{&#34;GyねんMがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;GyねんMがつdにち(E)～dにち(E)&#34;}
&#43;                    y{&#34;GyねんMがつdにち(E)～yねんMがつdにち(E)&#34;}
                 }
                 GyMMMd{
-                    G{&#34;Gy年M月d日～Gy年M月d日&#34;}
-                    M{&#34;Gy年M月d日～M月d日&#34;}
-                    d{&#34;Gy年M月d日～d日&#34;}
-                    y{&#34;Gy年M月d日～y年M月d日&#34;}
&#43;                    G{&#34;GyねんMがつdにち～GyねんMがつdにち&#34;}
&#43;                    M{&#34;GyねんMがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;GyねんMがつdにち～dにち&#34;}
&#43;                    y{&#34;GyねんMがつdにち～yねんMがつdにち&#34;}
                 }
                 GyMd{
                     G{&#34;GGGGGy/MM/dd～GGGGGy/MM/dd&#34;}
@@ -1299,32 &#43;1299,32 @@
                     H{&#34;H時～H時(v)&#34;}
                 }
                 M{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MEd{
                     M{&#34;MM/dd(E)～MM/dd(E)&#34;}
                     d{&#34;MM/dd(E)～MM/dd(E)&#34;}
                 }
                 MMM{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MMMEd{
-                    M{&#34;M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;M月d日(E)～d日(E)&#34;}
&#43;                    M{&#34;Mがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;Mがつdにち(E)～dにち(E)&#34;}
                 }
                 MMMM{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MMMd{
-                    M{&#34;M月d日～M月d日&#34;}
-                    d{&#34;M月d日～d日&#34;}
&#43;                    M{&#34;Mがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;Mがつdにち～dにち&#34;}
                 }
                 Md{
                     M{&#34;MM/dd～MM/dd&#34;}
                     d{&#34;MM/dd～MM/dd&#34;}
                 }
                 d{
-                    d{&#34;d日～d日&#34;}
&#43;                    d{&#34;dにち～dにち&#34;}
                 }
                 fallback{&#34;{0}～{1}&#34;}
                 h{
@@ -1346,7 &#43;1346,7 @@
                     h{&#34;aK時～K時(v)&#34;}
                 }
                 y{
-                    y{&#34;Gy年～y年&#34;}
&#43;                    y{&#34;Gyねん～yねん&#34;}
                 }
                 yM{
                     M{&#34;GGGGGy/MM～y/MM&#34;}
@@ -1358,22 &#43;1358,22 @@
                     y{&#34;GGGGGy/MM/dd(E)～y/MM/dd(E)&#34;}
                 }
                 yMMM{
-                    M{&#34;Gy年M月～M月&#34;}
-                    y{&#34;Gy年M月～y年M月&#34;}
&#43;                    M{&#34;GyねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;GyねんMがつ～yねんMがつ&#34;}
                 }
                 yMMMEd{
-                    M{&#34;Gy年M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;Gy年M月d日(E)～d日(E)&#34;}
-                    y{&#34;Gy年M月d日(E)～y年M月d日(E)&#34;}
&#43;                    M{&#34;GyねんMがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;GyねんMがつdにち(E)～dにち(E)&#34;}
&#43;                    y{&#34;GyねんMがつdにち(E)～yねんMがつdにち(E)&#34;}
                 }
                 yMMMM{
-                    M{&#34;Gy年M月～M月&#34;}
-                    y{&#34;Gy年M月～y年M月&#34;}
&#43;                    M{&#34;GyねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;GyねんMがつ～yねんMがつ&#34;}
                 }
                 yMMMd{
-                    M{&#34;Gy年M月d日～M月d日&#34;}
-                    d{&#34;Gy年M月d日～d日&#34;}
-                    y{&#34;Gy年M月d日～y年M月d日&#34;}
&#43;                    M{&#34;GyねんMがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;GyねんMがつdにち～dにち&#34;}
&#43;                    y{&#34;GyねんMがつdにち～yねんMがつdにち&#34;}
                 }
                 yMd{
                     M{&#34;GGGGGy/MM/dd～y/MM/dd&#34;}
@@ -1400,8 &#43;1400,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;y年M月d日EEEE&#34;,
-                &#34;y年M月d日&#34;,
&#43;                &#34;yねんMがつdにちEEEE&#34;,
&#43;                &#34;yねんMがつdにち&#34;,
                 &#34;y/MM/dd&#34;,
                 &#34;y/MM/dd&#34;,
                 &#34;{1} {0}&#34;,
@@ -1420,56 &#43;1420,56 @@
                 E{&#34;ccc&#34;}
                 EBhm{&#34;BK:mm (E)&#34;}
                 EBhms{&#34;BK:mm:ss (E)&#34;}
-                EEEEd{&#34;d日EEEE&#34;}
&#43;                EEEEd{&#34;dにちEEEE&#34;}
                 EHm{&#34;H:mm (E)&#34;}
                 EHms{&#34;H:mm:ss (E)&#34;}
-                Ed{&#34;d日(E)&#34;}
&#43;                Ed{&#34;dにち(E)&#34;}
                 Ehm{&#34;aK:mm (E)&#34;}
                 Ehms{&#34;aK:mm:ss (E)&#34;}
-                Gy{&#34;Gy年&#34;}
-                GyMMM{&#34;Gy年M月&#34;}
-                GyMMMEEEEd{&#34;Gy年M月d日EEEE&#34;}
-                GyMMMEd{&#34;Gy年M月d日(E)&#34;}
-                GyMMMd{&#34;Gy年M月d日&#34;}
&#43;                Gy{&#34;Gyねん&#34;}
&#43;                GyMMM{&#34;GyねんMがつ&#34;}
&#43;                GyMMMEEEEd{&#34;GyねんMがつdにちEEEE&#34;}
&#43;                GyMMMEd{&#34;GyねんMがつdにち(E)&#34;}
&#43;                GyMMMd{&#34;GyねんMがつdにち&#34;}
                 H{&#34;H時&#34;}
                 Hm{&#34;H:mm&#34;}
                 Hms{&#34;H:mm:ss&#34;}
                 Hmsv{&#34;H:mm:ss v&#34;}
                 Hmv{&#34;H:mm v&#34;}
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEEEEd{&#34;M/dEEEE&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEEEEd{&#34;M月d日EEEE&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEEEEd{&#34;MがつdにちEEEE&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
                 MMMMW{
-                    other{&#34;M月第W週&#34;}
&#43;                    other{&#34;Mがつ第W週&#34;}
                 }
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
&#43;                d{&#34;dにち&#34;}
                 h{&#34;aK時&#34;}
                 hm{&#34;aK:mm&#34;}
                 hms{&#34;aK:mm:ss&#34;}
                 hmsv{&#34;aK:mm:ss v&#34;}
                 hmv{&#34;aK:mm v&#34;}
                 ms{&#34;mm:ss&#34;}
-                y{&#34;y年&#34;}
&#43;                y{&#34;yねん&#34;}
                 yM{&#34;y/M&#34;}
                 yMEEEEd{&#34;y/M/dEEEE&#34;}
                 yMEd{&#34;y/M/d(E)&#34;}
                 yMM{&#34;y/MM&#34;}
-                yMMM{&#34;y年M月&#34;}
-                yMMMEEEEd{&#34;y年M月d日EEEE&#34;}
-                yMMMEd{&#34;y年M月d日(E)&#34;}
-                yMMMM{&#34;y年M月&#34;}
-                yMMMd{&#34;y年M月d日&#34;}
&#43;                yMMM{&#34;yねんMがつ&#34;}
&#43;                yMMMEEEEd{&#34;yねんMがつdにちEEEE&#34;}
&#43;                yMMMEd{&#34;yねんMがつdにち(E)&#34;}
&#43;                yMMMM{&#34;yねんMがつ&#34;}
&#43;                yMMMd{&#34;yねんMがつdにち&#34;}
                 yMd{&#34;y/M/d&#34;}
                 yQQQ{&#34;y/QQQ&#34;}
-                yQQQQ{&#34;y年QQQQ&#34;}
&#43;                yQQQQ{&#34;yねんQQQQ&#34;}
                 yw{
-                    other{&#34;Y年第w週&#34;}
&#43;                    other{&#34;yねん第w週&#34;}
                 }
             }
             dayNames{
@@ -1649,8 &#43;1649,8 @@
                     m{&#34;BK:mm～K:mm&#34;}
                 }
                 Gy{
-                    G{&#34;Gy年～Gy年&#34;}
-                    y{&#34;Gy年～y年&#34;}
&#43;                    G{&#34;Gyねん～Gyねん&#34;}
&#43;                    y{&#34;Gyねん～yねん&#34;}
                 }
                 GyM{
                     G{&#34;Gy/MM～Gy/MM&#34;}
@@ -1664,21 &#43;1664,21 @@
                     y{&#34;Gy/MM/dd(E)～y/MM/dd(E)&#34;}
                 }
                 GyMMM{
-                    G{&#34;Gy年M月～Gy年M月&#34;}
-                    M{&#34;Gy年M月～M月&#34;}
-                    y{&#34;Gy年M月～y年M月&#34;}
&#43;                    G{&#34;GyねんMがつ～GyねんMがつ&#34;}
&#43;                    M{&#34;GyねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;GyねんMがつ～yねんMがつ&#34;}
                 }
                 GyMMMEd{
-                    G{&#34;Gy年M月d日(E)～Gy年M月d日(E)&#34;}
-                    M{&#34;Gy年M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;Gy年M月d日(E)～d日(E)&#34;}
-                    y{&#34;Gy年M月d日(E)～y年M月d日(E)&#34;}
&#43;                    G{&#34;GyねんMがつdにち(E)～GyねんMがつdにち(E)&#34;}
&#43;                    M{&#34;GyねんMがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;GyねんMがつdにち(E)～dにち(E)&#34;}
&#43;                    y{&#34;GyねんMがつdにち(E)～yねんMがつdにち(E)&#34;}
                 }
                 GyMMMd{
-                    G{&#34;Gy年M月d日～Gy年M月d日&#34;}
-                    M{&#34;Gy年M月d日～M月d日&#34;}
-                    d{&#34;Gy年M月d日～d日&#34;}
-                    y{&#34;Gy年M月d日～y年M月d日&#34;}
&#43;                    G{&#34;GyねんMがつdにち～GyねんMがつdにち&#34;}
&#43;                    M{&#34;GyねんMがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;GyねんMがつdにち～dにち&#34;}
&#43;                    y{&#34;GyねんMがつdにち～yねんMがつdにち&#34;}
                 }
                 GyMd{
                     G{&#34;Gy/MM/dd～Gy/MM/dd&#34;}
@@ -1701,32 &#43;1701,32 @@
                     H{&#34;H時～H時(v)&#34;}
                 }
                 M{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MEd{
                     M{&#34;MM/dd(E)～MM/dd(E)&#34;}
                     d{&#34;MM/dd(E)～MM/dd(E)&#34;}
                 }
                 MMM{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MMMEd{
-                    M{&#34;M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;M月d日(E)～d日(E)&#34;}
&#43;                    M{&#34;Mがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;Mがつdにち(E)～dにち(E)&#34;}
                 }
                 MMMM{
-                    M{&#34;M月～M月&#34;}
&#43;                    M{&#34;Mがつ～Mがつ&#34;}
                 }
                 MMMd{
-                    M{&#34;M月d日～M月d日&#34;}
-                    d{&#34;M月d日～d日&#34;}
&#43;                    M{&#34;Mがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;Mがつdにち～dにち&#34;}
                 }
                 Md{
                     M{&#34;MM/dd～MM/dd&#34;}
                     d{&#34;MM/dd～MM/dd&#34;}
                 }
                 d{
-                    d{&#34;d日～d日&#34;}
&#43;                    d{&#34;dにち～dにち&#34;}
                 }
                 fallback{&#34;{0}～{1}&#34;}
                 h{
@@ -1748,7 &#43;1748,7 @@
                     h{&#34;aK時～K時(v)&#34;}
                 }
                 y{
-                    y{&#34;y年～y年&#34;}
&#43;                    y{&#34;yねん～yねん&#34;}
                 }
                 yM{
                     M{&#34;y/MM～y/MM&#34;}
@@ -1760,22 &#43;1760,22 @@
                     y{&#34;y/MM/dd(E)～y/MM/dd(E)&#34;}
                 }
                 yMMM{
-                    M{&#34;y年M月～M月&#34;}
-                    y{&#34;y年M月～y年M月&#34;}
&#43;                    M{&#34;yねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;yねんMがつ～yねんMがつ&#34;}
                 }
                 yMMMEd{
-                    M{&#34;y年M月d日(E)～M月d日(E)&#34;}
-                    d{&#34;y年M月d日(E)～d日(E)&#34;}
-                    y{&#34;y年M月d日(E)～y年M月d日(E)&#34;}
&#43;                    M{&#34;yねんMがつdにち(E)～Mがつdにち(E)&#34;}
&#43;                    d{&#34;yねんMがつdにち(E)～dにち(E)&#34;}
&#43;                    y{&#34;yねんMがつdにち(E)～yねんMがつdにち(E)&#34;}
                 }
                 yMMMM{
-                    M{&#34;y年M月～M月&#34;}
-                    y{&#34;y年M月～y年M月&#34;}
&#43;                    M{&#34;yねんMがつ～Mがつ&#34;}
&#43;                    y{&#34;yねんMがつ～yねんMがつ&#34;}
                 }
                 yMMMd{
-                    M{&#34;y年M月d日～M月d日&#34;}
-                    d{&#34;y年M月d日～d日&#34;}
-                    y{&#34;y年M月d日～y年M月d日&#34;}
&#43;                    M{&#34;yねんMがつdにち～Mがつdにち&#34;}
&#43;                    d{&#34;yねんMがつdにち～dにち&#34;}
&#43;                    y{&#34;yねんMがつdにち～yねんMがつdにち&#34;}
                 }
                 yMd{
                     M{&#34;y/MM/dd～y/MM/dd&#34;}
@@ -1922,8 &#43;1922,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;Gy年M月d日EEEE&#34;,
-                &#34;Gy年M月d日&#34;,
&#43;                &#34;GyねんMがつdにちEEEE&#34;,
&#43;                &#34;GyねんMがつdにち&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;{1} {0}&#34;,
@@ -2155,8 &#43;2155,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;Gy年M月d日EEEE&#34;,
-                &#34;Gy年M月d日&#34;,
&#43;                &#34;GyねんMがつdにちEEEE&#34;,
&#43;                &#34;GyねんMがつdにち&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;{1} {0}&#34;,
@@ -2166,15 &#43;2166,15 @@
                 &#34;{1} {0}&#34;,
             }
             availableFormats{
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
-                y{&#34;Gy年&#34;}
&#43;                d{&#34;dにち&#34;}
&#43;                y{&#34;Gyねん&#34;}
             }
             eras{
                 abbreviated{
@@ -2285,15 &#43;2285,15 @@
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
                 {
-                    &#34;Gy年M月d日EEEE&#34;,
&#43;                    &#34;GyねんMがつdにちEEEE&#34;,
                     &#34;y=jpanyear&#34;,
                 }
                 {
-                    &#34;Gy年M月d日&#34;,
&#43;                    &#34;GyねんMがつdにち&#34;,
                     &#34;y=jpanyear&#34;,
                 }
                 {
-                    &#34;Gy年M月d日&#34;,
&#43;                    &#34;GyねんMがつdにち&#34;,
                     &#34;y=jpanyear&#34;,
                 }
                 &#34;GGGGGy/M/d&#34;,
@@ -2305,44 &#43;2305,44 @@
             }
             availableFormats{
                 E{&#34;ccc&#34;}
-                EEEEd{&#34;d日EEEE&#34;}
-                Ed{&#34;d日(E)&#34;}
-                Gy{&#34;Gy年&#34;}
-                GyMMM{&#34;Gy年M月&#34;}
-                GyMMMEEEEd{&#34;Gy年M月d日EEEE&#34;}
-                GyMMMEd{&#34;Gy年M月d日(E)&#34;}
-                GyMMMd{&#34;Gy年M月d日&#34;}
&#43;                EEEEd{&#34;dにちEEEE&#34;}
&#43;                Ed{&#34;dにち(E)&#34;}
&#43;                Gy{&#34;Gyねん&#34;}
&#43;                GyMMM{&#34;GyねんMがつ&#34;}
&#43;                GyMMMEEEEd{&#34;GyねんMがつdにちEEEE&#34;}
&#43;                GyMMMEd{&#34;GyねんMがつdにち(E)&#34;}
&#43;                GyMMMd{&#34;GyねんMがつdにち&#34;}
                 H{&#34;H時&#34;}
                 Hm{&#34;H:mm&#34;}
                 Hms{&#34;H:mm:ss&#34;}
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEEEEd{&#34;M/dEEEE&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEEEEd{&#34;M月d日EEEE&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEEEEd{&#34;MがつdにちEEEE&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
&#43;                d{&#34;dにち&#34;}
                 h{&#34;aK時&#34;}
                 hm{&#34;aK:mm&#34;}
                 hms{&#34;aK:mm:ss&#34;}
                 ms{&#34;mm:ss&#34;}
-                y{&#34;Gy年&#34;}
-                yyyy{&#34;Gy年&#34;}
&#43;                y{&#34;Gyねん&#34;}
&#43;                yyyy{&#34;Gyねん&#34;}
                 yyyyM{&#34;GGGGGy/M&#34;}
                 yyyyMEEEEd{&#34;GGGGGy/M/dEEEE&#34;}
                 yyyyMEd{&#34;GGGGGy/M/d(E)&#34;}
                 yyyyMM{&#34;GGGGGy/MM&#34;}
-                yyyyMMM{&#34;Gy年M月&#34;}
-                yyyyMMMEEEEd{&#34;Gy年M月d日EEEE&#34;}
-                yyyyMMMEd{&#34;Gy年M月d日(E)&#34;}
-                yyyyMMMM{&#34;Gy年M月&#34;}
-                yyyyMMMd{&#34;Gy年M月d日&#34;}
&#43;                yyyyMMM{&#34;GyねんMがつ&#34;}
&#43;                yyyyMMMEEEEd{&#34;GyねんMがつdにちEEEE&#34;}
&#43;                yyyyMMMEd{&#34;GyねんMがつdにち(E)&#34;}
&#43;                yyyyMMMM{&#34;GyねんMがつ&#34;}
&#43;                yyyyMMMd{&#34;GyねんMがつdにち&#34;}
                 yyyyMd{&#34;GGGGGy/M/d&#34;}
                 yyyyQQQ{&#34;Gy/QQQ&#34;}
-                yyyyQQQQ{&#34;Gy年QQQQ&#34;}
&#43;                yyyyQQQQ{&#34;GyねんQQQQ&#34;}
             }
             eras{
                 abbreviated{
@@ -2934,8 &#43;2934,8 @@
                 &#34;H:mm:ss z&#34;,
                 &#34;H:mm:ss&#34;,
                 &#34;H:mm&#34;,
-                &#34;Gy年M月d日EEEE&#34;,
-                &#34;Gy年M月d日&#34;,
&#43;                &#34;GyねんMがつdにちEEEE&#34;,
&#43;                &#34;GyねんMがつdにち&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;Gy/MM/dd&#34;,
                 &#34;{1} {0}&#34;,
@@ -2945,15 &#43;2945,15 @@
                 &#34;{1} {0}&#34;,
             }
             availableFormats{
-                M{&#34;M月&#34;}
&#43;                M{&#34;Mがつ&#34;}
                 MEd{&#34;M/d(E)&#34;}
-                MMM{&#34;M月&#34;}
-                MMMEd{&#34;M月d日(E)&#34;}
-                MMMMd{&#34;M月d日&#34;}
-                MMMd{&#34;M月d日&#34;}
&#43;                MMM{&#34;Mがつ&#34;}
&#43;                MMMEd{&#34;Mがつdにち(E)&#34;}
&#43;                MMMMd{&#34;Mがつdにち&#34;}
&#43;                MMMd{&#34;Mがつdにち&#34;}
                 Md{&#34;M/d&#34;}
-                d{&#34;d日&#34;}
-                y{&#34;Gy年&#34;}
&#43;                d{&#34;dにち&#34;}
&#43;                y{&#34;Gyねん&#34;}
             }
             eras{
                 abbreviated{
@@ -3069,11 &#43;3069,11 @@
         day{
             dn{&#34;日&#34;}
             relative{
-                &#34;-1&#34;{&#34;昨日&#34;}
-                &#34;-2&#34;{&#34;一昨日&#34;}
-                &#34;0&#34;{&#34;今日&#34;}
-                &#34;1&#34;{&#34;明日&#34;}
-                &#34;2&#34;{&#34;明後日&#34;}
&#43;                &#34;-1&#34;{&#34;きのう&#34;}
&#43;                &#34;-2&#34;{&#34;おととい&#34;}
&#43;                &#34;0&#34;{&#34;きょう&#34;}
&#43;                &#34;1&#34;{&#34;あした&#34;}
&#43;                &#34;2&#34;{&#34;あさって&#34;}
             }
             relativeTime{
                 future{
@@ -3087,11 &#43;3087,11 @@
         day-narrow{
             dn{&#34;日&#34;}
             relative{
-                &#34;-1&#34;{&#34;昨日&#34;}
-                &#34;-2&#34;{&#34;一昨日&#34;}
-                &#34;0&#34;{&#34;今日&#34;}
-                &#34;1&#34;{&#34;明日&#34;}
-                &#34;2&#34;{&#34;明後日&#34;}
&#43;                &#34;-1&#34;{&#34;きのう&#34;}
&#43;                &#34;-2&#34;{&#34;おととい&#34;}
&#43;                &#34;0&#34;{&#34;きょう&#34;}
&#43;                &#34;1&#34;{&#34;あした&#34;}
&#43;                &#34;2&#34;{&#34;あさって&#34;}
             }
             relativeTime{
                 future{
@@ -3105,11 &#43;3105,11 @@
         day-short{
             dn{&#34;日&#34;}
             relative{
-                &#34;-1&#34;{&#34;昨日&#34;}
-                &#34;-2&#34;{&#34;一昨日&#34;}
-                &#34;0&#34;{&#34;今日&#34;}
-                &#34;1&#34;{&#34;明日&#34;}
-                &#34;2&#34;{&#34;明後日&#34;}
&#43;                &#34;-1&#34;{&#34;きのう&#34;}
&#43;                &#34;-2&#34;{&#34;おととい&#34;}
&#43;                &#34;0&#34;{&#34;きょう&#34;}
&#43;                &#34;1&#34;{&#34;あした&#34;}
&#43;                &#34;2&#34;{&#34;あさって&#34;}
             }
             relativeTime{
                 future{

```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/%E5%A2%9E%E5%8A%A0%E8%AF%AD%E8%A8%80%E6%96%B9%E6%A1%88/  

