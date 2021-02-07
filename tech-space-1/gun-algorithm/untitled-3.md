# 字符串排序

### 背景

字符串排序在实际的使用中通常会对应以下的几种情况：搜索系统，对于繁多的搜索结果需要进行一定的排序；通信系统，无论何种信息，都是一堆字符从一端传递到另一端；基因工程，出人意料的是，在生物学中有着十分重要的作用，比如碱基的配对。

### 键索引计数法

在引入字符串排序的方法之前，我们先考察一种叫键索引计数法的方法。比如说有一群分别属于不同组的学生，如下

```text
name    no
Anderw    1
Jack    2
Peter    1
Jane    3
Janie    2
...        ...
```

我们的目的在于将学生按照组别进行排序。采用的方法可以采用以下几个步骤：

#### 频率统计

首先遍历一遍整个字符串，使用一个额外的统计数组来记录每个组别开始的位置，这里有一个小技巧，如果组别有5组，那么这个统计数组的大小设置为6。我们将这个统计数组命名为count。组别为r的记录将在r+1位置的count进行加1。

```cpp
for(int i=0;i<N;i++)
    count[a[i].key()+1]++;
```

#### 将频率转换为索引

接下来就是将上述的频率转换为排序结果中的索引位置。这样的话，上述的count数组的巧妙地小技巧就十分有用。

```cpp
for(int r=0;r<N;++r){
    count[r+1]+=count[r]
}
```

#### 数据分类

在确定了每一组的起始索引之后，我们需要做的就是对数组进行一个分类。这里会使用到一个额外的数组空间，分类的过程可以理解为多指针的过程。

```cpp
for(int i=0;i<N;i++){
    aux[count[a[i].key()]++]=a[i];
}
```

### LSD 低位优先的字符串排序

LSD由字面意思可得，是从字符串的低位开始。LSD的局限在于，他排序的字符串必须是等长的。这种情况实际上是很常见的，比如排序车牌、ip地址等。

LSD的具体步骤是，从右至左开始，对每一位实施键索引计数法。如果这个字符串长度为W，那么实际上就是实施W次键索引计数法。并且这种排序方法是稳定的。

```cpp
void LSDSort(vector<string> a,int len){
    int N=a.size();
    int R=256;
    vector<string> aux(N);
    for(int d=W-1;d>=0;--d){
        vector<int> count(R+1);
        for(int i=0;i<N;i++)    //计算出现的频率
            count[a[i][d]+1]++;
        for(int i=0;i<R;i++)    //将频率转换为索引
            count[i+1]+=count[i];
        for(int i=0;i<N;i++)    //分类元素
            aux[count[a[i][d]]++]=a[i];
        for(int i=0;i<N;i++)    //回写
            a[i]=aux[i];

    }
}
```

### MSD 高位优先的字符串排序

高位优先的排序可以适用于随机的字符串。其本质是从高位对字符串使用递归的键索引计数法。 基本的代码逻辑如下所示

```cpp
class MSD{
private:
    int R = 256;    //基数
    int M = 15;     //小数组的切换阈值
    string aux;     //数据分类的辅助数组

    int charat(string s,int d){
        if(d<s.size())  return s[d];
        else    return -1;
    }

public:
    void sort(string s){
        int n=s.size();
        aux==string(n);
        sort(a,0,n-1,0);
    }

    //以第d个字符为键将a[low]至a[high]排序
    void sort(string s,int low,int high,int d){
        if(high<=low+M){
            InsertionSort(a,low,high,d);
            return;
        }
        vector<int> count(R+2);
        for(int i=low;i<=high;i++)  //计算频率
            count[charat(a[i],d)+2]++;
        for(int r=0;r<R+1;r++)      //将频率转换为索引
            count[r+1]+=count[r];
        for(int i=low;i<=high;i++)  //数据分类
            aux[count[charat(a[i],d)+1]++]=a[i];
        for(int i=low;i<=high;i++)  //回写
            a[i]=aux[i-low];

        //递归的以每个字符为键进行排序
        for(int r=0;r<R;r++)
            sort(a,low+count[r],low+count[r+1]-1,d+1);

    }
}
```

同样，因为MSD所使用的算法基本上是使用快速排序，因此他所面临的问题也是快速排序所面临的问题。

#### 小型子数组

对于小型子数组，可以不使用快速排序，这样会产生大量的小型数组，但事实上这样速度会慢很多。因此我们可以设定一个阈值，在该阈值之下使用插入排序。

#### 等值键

第二个陷阱是，对于大量的等值键的数组排序会变慢。高位优先字符串的最坏情况就是所有的键都相同。

### 三向字符串快速排序

我们可以根据高位优先的字符串排序改进快速排序，根据键的首字母进行三向切分，仅在中间子数组的下一个字符继续进行递归排序。

```cpp
class Quick3string{
    int charat(string s,int d){
      if(d<s.size())  return s[d];
      else    return -1;
   }
    void sort(string s){
        sort(a,0,a.size()-1,0);
    }
    void sort(string s,int low,int high,int d){
        if(high<=low) return;
        int lt=low,gt=high;
        int v=charat(a[low],d);
        int i=low+1;
        while(i<=gt){
            int t=charat(a[i],d);
            if(t<v) swap(a[lt++],a[i++]);
            else if(t<v) swap(a[i],a[gt—]);
            else    i++;
        }
        sort(a,low,lt-1,d);
        if(v>=0)    sort(a,lt,gt,d+1);
        sort(a,gt+1,high,d);
    }
}
```

