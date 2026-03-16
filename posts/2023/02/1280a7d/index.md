# 博客图片优化

本篇文章介绍了如何优化博客中使用的图片。

<!--more-->

## convert

在 macos上 安装 imagemagick。imagemagick 是一个开源图片工具，支持对图片的大量操作。

```sh
brew install imagemagick
```

### 改变大小(resize)

```sh

## 缩小为 50%大小
convert image.png -resize 50% image2.png

## 固定宽度
convert original-image.jpg -resize 100x converted-image.jpg

## 固定高度
convert original-image.jpg -resize x100 converted-image.jpg

## 固定尺寸
convert original-image.jpg -resize 100x100 converted-image.jpg
```

### 旋转(rotate)

```sh
## 旋转45度
convert flower.jpg -rotate 45 flower_rotate45.jpg
```

### 图片质量

```sh
## Convert with 70% quality.
convert -quality 70 image.png new_image.png
```

### 改变文件格式

```sh
## To convert a file from jpg to pdf
convert original.jpg converted.pdf
## Convert an image from JPG to PNG:
convert image.jpg image.png
```

### 拼接

```sh
## Horizontally append images:
convert image1.png image2.png image3.png +append image123.png

## Vertically append images:
convert image1.png image2.png image3.png -append image123.png
```

### 组合

```sh
## Convert and combine multiple images to a single PDF.
convert image1.png image2.jpg image3.bmp output.pdf

## Create a GIF from a series of images with 100ms delay between them:
convert image1.png image2.png image3.png -delay 10 animation.gif

## Create a favicon from several images of different sizes:
convert image1.png image2.png image3.png image.ico
```

### 分页

```sh
## To convert an N page pdf to N images (will autonumber):
convert -density 150 arch1.pdf -quality 80 'output.jpg'

## To convert an N page pdf to N images with explicit filename formatting:
convert -density 150 arch1.pdf -quality 80 'output-%d.jpg'
```

### 遍历文件夹

```sh
## To resize all of the images within a directory:
for file in `ls original/image/path/`;
    do new_path=${file%.*};
    new_file=`basename $new_path`;
    convert $file -resize 150 converted/image/path/$new_file.png;
done
```

## cwebp

webp 是一种新的图像格式，用于web项目，可以大大提高网站访问速度。

- 同样的分辨率，大小比 jpg、png 小 25% 以上；
- Chrome、Firefox、Edge、Opera 等都支持此格式。

在macos 上安装:

```sh
brew install webp
```

### cwebp

将图像 转换为 WebP，支持的格式有：PNG、JPEG、TIFF、WebP 等。

```sh
cwebp -q 100 image.png -o image.webp
```
### dwebp

将 WebP 转换为图像，支持的格式有：为 JPE、PNG、PAM、PPM 或 PGM 图像。

```sh
dwebp picture.webp -o output.png
```

### gif2webp

将 GIF 图像转换为 WebP。

```
gif2webp picture.gif -o picture.webp
```

### img2webp

从输入图像序列创建动画 WebP 文件，支持的格式有：PNG、JPEG、TIFF 或 WebP。

```
img2webp -loop 2 in0.png -lossy in1.jpg -d 80 in2.tiff -o out.webp
```

### vwebp

解压缩 WebP 文件并使用 OpenGL 在窗口中显示它。

```
vwebp picture.webp
```

### 批量处理

```sh
find <dir path> -name "*.png" -exec cwebp {} -o {}.webp;
```

### 递归处理脚本

```sh
#!/bin/bash
## shell script: convert2webp.sh
## convert *.png,*.jpg,*.jepg to webp
## usage: bash convert2webp.sh <dir path> -r 

function travelDir()
{
  files=$(ls "$1")
  for file in $files
  do
    if [ -d "$1/$file" ]; then
        safeMk "$2/$file"
        travelDir "$1/$file" "$2/$file"
    else
        convert2webp "$1/$file" "$2"
    fi
  done
}

function handleDir()
{
  files=$(ls "$1")
  for file in $files
  do
    if [ ! -d "$1/$file" ]; then
      convert2webp "$1/$file" "$2"
    fi
  done
}

function convert2webp()
{
  filename=$1
  trash=$2
  echo "handle file: $filename"
  fileType="${filename##*.}"
  low=$(echo "$fileType" | tr '[:upper:]' '[:lower:]')
  name="${filename%.*}"
  if [ "$low"x = "pngx" ] || [ "$low"x = "jpgx" ] ||  [ "$low"x = "jpegx" ]; then
    target=$name'.webp'
    echo "being convert $filename to $target"
    cwebp "$filename" -o "$target"
    mv "$filename" "$trash"
  fi
}

function safeMk()
{
  if [ ! -d "$1" ]; then
    mkdir "$1"
  fi
}

if [ ! -d "$1" ];then
  echo "please input a dir"
  exit 1
fi
if [ -n "$2" ] && [ ! "$2"x = "-rx" ]; then
  echo "use -r or input nothing"
  exit 1
fi


type cwebp
if [ $? -ne 0 ]; then
  echo "please install cwebp, MacOS can use 'brew install webp', ubuntu use 'apt install webp'"
  exit 1
fi

type trash
if [ $? -ne 0 ]; then
  echo "please install trash, MacOS can use 'brew install trash', ubuntu use 'apt install trash-cli'"
  exit 1
fi

workdir=$(cd $(dirname "$0") || exit; pwd)
path=$1
cd "$path" || exit
safeMk ".webptrash"

if [ "$2"x = "-rx" ]; then
  # -r 递归
  travelDir "." ".webptrash"
else
  handleDir "." ".webptrash"
fi

tree -a
trash .webptrash

cd "$workdir" || exit
```

