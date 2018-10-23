# 使用CUDA加速实现真彩色BMP图片转灰度图

标签（空格分隔）： CUDA

---
[TOC]

## BMP图相关知识

### 1. 简介

BMP（全称Bitmap）是Windows操作系统中的标准图像文件格式，可以分成两类：**设备相关位图**（DDB）和**设备无关位图**（DIB），使用非常广。它采用位映射存储格式，除了图像深度可选以外，**不采用其他任何压缩**，因此，BMP文件所占用的空间很大。BMP文件的图像深度可选lbit、4bit、8bit及24bit。BMP文件存储数据时，图像的扫描方式是按**从左到右、从下到上**的顺序。由于BMP文件格式是Windows环境中交换与图有关的数据的一种标准，因此在Windows环境中运行的图形图像软件都支持BMP图像格式。

### 2. 格式组成

**典型的BMP图像文件由四部分组成：**

- 位图头文件数据结构，它包含BMP图像文件的类型、显示内容等信息。
**其结构图定义如下**
```
typedef struct tagBITMAPFILEHEADER
{
    WORD bfType;//位图文件的类型，必须为BM(1-2字节）
    DWORD bfSize;//位图文件的大小，以字节为单位（3-6字节，低位在前）
    WORD bfReserved1;//位图文件保留字，必须为0(7-8字节）
    WORD bfReserved2;//位图文件保留字，必须为0(9-10字节）
    DWORD bfOffBits;//位图数据的起始位置，以相对于位图（11-14字节，低位在前）
    //文件头的偏移量表示，以字节为单位
}__attribute__((packed)) BITMAPFILEHEADER;
```
- 位图信息数据结构，它包含有BMP图像的宽、高、压缩方法，以及定义颜色等信息。
**其结构图定义如下**
```
typedef struct tagBITMAPINFOHEADER{
DWORD biSize;//本结构所占用字节数（15-18字节）
LONG biWidth;//位图的宽度，以像素为单位（19-22字节）
LONG biHeight;//位图的高度，以像素为单位（23-26字节）
WORD biPlanes;//目标设备的级别，必须为1(27-28字节）
WORD biBitCount;//每个像素所需的位数，必须是1（双色），（29-30字节）
//4(16色），8(256色）16(高彩色)或24（真彩色）之一
DWORD biCompression;//位图压缩类型，必须是0（不压缩），（31-34字节）
//1(BI_RLE8压缩类型）或2(BI_RLE4压缩类型）之一
DWORD biSizeImage;//位图的大小(其中包含了为了补齐行数是4的倍数而添加的空字节)，以字节为单位（35-38字节）
LONG biXPelsPerMeter;//位图水平分辨率，每米像素数（39-42字节）
LONG biYPelsPerMeter;//位图垂直分辨率，每米像素数（43-46字节)
DWORD biClrUsed;//位图实际使用的颜色表中的颜色数（47-50字节）
DWORD biClrImportant;//位图显示过程中重要的颜色数（51-54字节）
}__attribute__((packed)) BITMAPINFOHEADER;
```
- 调色板，这个部分是可选的，有些位图需要调色板，有些位图，比如真彩色图（24位的BMP）就不需要调色板.**（由于本次实验使用真彩色图因而不需要定义调色板）**
- 位图数据，这部分的内容根据BMP位图使用的位数不同而不同，在**24位图中直接使用RGB**，而其他的小于24位的使用调色板中颜色索引值。位图数据记录了位图的每一个像素值，记录顺序是在扫描**行内是从左到右**，扫描**行间是从下到上**。

### 3. 真彩色BMP图

真彩色BMP中的位图数据直接采用RGB值定义，即**一个像素点用Red、Green、Blue三个分量**定义，每个分量的**取值范围从0-255**共计256种取值。

## 真彩色图与灰度图的转变

### 1. 转换原理
$$(Red,Green,Blue)-->(Gray,Gray,Gray)$$
在全彩色图中，每个像素都由RGB三个分量构成，如果按照一定的算法，对RGB值进行处理，得到灰度值（Gray），将灰度值赋给RGB三个分量就可以得到该全彩色图的灰度图。

### 2. 转换算法

- 平均法
$$Gray = \frac{(Red + Green + Blue)} {3}$$ 
这个算法可以生成不错灰度值，因为公式简单，所以易于维护和优化。然而它也不是没有缺点，因为简单快速，从人眼的感知角度看，图片的灰度阴影和亮度方面做的还不够好。所以，我们需要更复杂的运算。

- 基于人眼感知的公式
$$Gray = Red*0.299 + Green*0.587 + Blue*0.114$$
人类对红绿蓝三色的感知程度依次是：绿>红>蓝，所以平均算法从这个角度看是不科学的。应该按照人类对光的感知程度为每个颜色设定一个权重，它们的之间的地位不应该是平等的。

- 去饱和法
$$Gray = \frac{Max(Red, Green, Blue) + Min(Red, Green, Blue)} {2}$$
去饱和的过程就是把RGB转换为HLS，然后将饱和度设为0。因此，我们需要取一种颜色，转换它为最不饱和的值。去饱和后，图片立体感减弱，但是更柔和。

### 3. 方法的选取
经过实际测试，发现使用基于人眼感知的公式产生的灰度图效果更好，有更好的层次感而且利于并行优化，故最终选取基于人眼感知的公式生成灰度图。

## 串行方式实现真彩色BMP图片转灰度图

### 1. 使用C语言结构体定义位图文件

#### 位图头文件数据结构
```c
typedef struct tagBITMAPFILEHEADER
{
    unsigned char bfType[2];//文件格式
    unsigned long bfSize;//文件大小
    unsigned short bfReserved1;//保留
    unsigned short bfReserved2;
    unsigned long bfOffBits; //DIB数据在文件中的偏移量
}fileHeader;
```

#### 位图信息数据结构
```c
typedef struct tagBITMAPINFOHEADER
{
    unsigned long biSize;//该结构的大小
    long biWidth;//文件宽度
    long biHeight;//文件高度
    unsigned short biPlanes;//平面数
    unsigned short biBitCount;//颜色位数
    unsigned long biCompression;//压缩类型
    unsigned long biSizeImage;//DIB数据区大小
    long biXPixPerMeter;
    long biYPixPerMeter;
    unsigned long biClrUsed;//多少颜色索引表
    unsigned long biClrImporant;//多少重要颜色
}fileInfo;
```

#### 调色板结构（真彩色图）
```c
typedef struct tagRGBQUAD
{
    unsigned char rgbBlue; //蓝色分量亮度
    unsigned char rgbGreen;//绿色分量亮度
    unsigned char rgbRed;//红色分量亮度
    unsigned char rgbReserved;
}rgbq;
```
### 2. 使用串行程序的完整代码（C语言）

```c
#include<stdio.h>
#include<malloc.h>
#include<stdlib.h>

/*
位图头结构
*/
#pragma pack(1)
typedef struct tagBITMAPFILEHEADER
{
    unsigned char bfType[2];//文件格式
    unsigned long bfSize;//文件大小
    unsigned short bfReserved1;//保留
    unsigned short bfReserved2;
    unsigned long bfOffBits; //DIB数据在文件中的偏移量
}fileHeader;
#pragma pack()

/*
位图数据信息结构
*/
#pragma pack(1)
typedef struct tagBITMAPINFOHEADER
{
    unsigned long biSize;//该结构的大小
    long biWidth;//文件宽度
    long biHeight;//文件高度
    unsigned short biPlanes;//平面数
    unsigned short biBitCount;//颜色位数
    unsigned long biCompression;//压缩类型
    unsigned long biSizeImage;//DIB数据区大小
    long biXPixPerMeter;
    long biYPixPerMeter;
    unsigned long biClrUsed;//多少颜色索引表
    unsigned long biClrImporant;//多少重要颜色
}fileInfo;
#pragma pack()

/*
调色板结构
*/
#pragma pack(1)
typedef struct tagRGBQUAD
{
    unsigned char rgbBlue; //蓝色分量亮度
    unsigned char rgbGreen;//绿色分量亮度
    unsigned char rgbRed;//红色分量亮度
    unsigned char rgbReserved;
}rgbq;
#pragma pack()

int main()
{
    /*存储RGB图像的一行像素点*/
    unsigned char ImgData[3000][3];
    /*将灰度图的像素存到一个一维数组中*/
    unsigned char ImgData2[3000];
    int i,j,k;
    FILE * fpBMP,* fpGray;
    fileHeader * fh;
    fileInfo * fi;
    rgbq * fq;
    char filename1[20],filename2[20];

    printf("输入图像文件名：");
    scanf("%s",filename1);
    if((fpBMP=fopen(filename1,"rb"))==NULL)
    {
        printf("打开文件失败");
        exit(0);
    }
    printf("输出图像文件名：");
    scanf("%s",filename2);
    if((fpGray=fopen(filename2,"wb"))==NULL)
    {
        printf("创建文件失败");
        exit(0);
    }

    fh=(fileHeader *)malloc(sizeof(fileHeader));
    fi=(fileInfo *)malloc(sizeof(fileInfo));
    //读取位图头结构和信息头
    fread(fh,sizeof(fileHeader),1,fpBMP);
    fread(fi,sizeof(fileInfo),1,fpBMP);
    //修改头信息
    fi->biBitCount=8;
    fi->biSizeImage=( (fi->biWidth*3+3)/4 ) * 4*fi->biHeight;
    //fi->biClrUsed=256;

    fh->bfOffBits = sizeof(fileHeader)+sizeof(fileInfo)+256*sizeof(rgbq);
    fh->bfSize = fh->bfOffBits + fi->biSizeImage;

    //创建调色版
    fq=(rgbq *)malloc(256*sizeof(rgbq));
    for(i=0;i<256;i++)
    {
        fq[i].rgbBlue=fq[i].rgbGreen=fq[i].rgbRed=i;
        //fq[i].rgbReserved=0;
    }
    //将头信息写入
    fwrite(fh,sizeof(fileHeader),1,fpGray);
    fwrite(fi,sizeof(fileInfo),1,fpGray);
    fwrite(fq,sizeof(rgbq),256,fpGray);

    /********算法核心********/
    //读取RGB图像素并转换为灰度值
    for ( i=0;i<fi->biHeight;i++ )
    {
        for(j=0;j<(fi->biWidth+3)/4*4;j++)
        {
            for(k=0;k<3;k++)
                fread(&ImgData[j][k],1,1,fpBMP);
        }
        for(j=0;j<(fi->biWidth+3)/4*4;j++)
        {
            ImgData2[j]=( ImgData[j][0] * 0.114 +
                        ImgData[j][1] * 0.587 +
                        ImgData[j][2] * 0.299 );
        }
        //将灰度图信息写入
        fwrite(ImgData2,j,1,fpGray);
    }
    /********算法核心********/

        free(fh);
        free(fi);
        free(fq);
        fclose(fpBMP);
        fclose(fpGray);
        printf("success\n");
        return 0;
}
```

### 3. 结果展示

样例使用长宽比为3000*2000的全彩色BMP图片进行测试（由于BMP图片是一种无压缩的图片格式，图片非常大。为了方便展示以下样例使用的图片已经转换为jpg格式）

#### 全彩色图像

![Sierra](https://raw.githubusercontent.com/huiyixiang/MarkdownPhotos/master/BMPtoGray/test2.jpg)

#### 灰度图像
![Sierra](https://raw.githubusercontent.com/huiyixiang/MarkdownPhotos/master/BMPtoGray/2.jpg)

### 4.用时分析


## 使用CUDA加速彩色图片转化灰度图片

### 
