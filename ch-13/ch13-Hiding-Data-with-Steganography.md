# 第13章：用隐写术隐藏数据

`steganography` 一词是希腊词 `steganos`（意为覆盖、隐藏或保护）和 `graphien` （意为书写）的组合。 在安全方面，`steganography`是指用于通过将数据植入其他数据（例如图像）中来混淆（或隐藏）数据的技术和程序，以便可以在未来的某个时间点提取数据。作为安全社区的一员，通过隐藏交付给目标后恢复的有效负载来研究这一常规实践。

在本章中，你将植入 Portable Network Graphics(PNG)图片的数据。首先研究PNG格式，并学习如何读取PNG数据。 然后将自己的数据植入到现存的图像中。最后，探索XOR，一种加解密植入数据的方法。

## 探索PNG格式

先从查看PNG说明开始，这有助于理解PNG图像的格式，及如何将数据植入到文件中。 在http://www.libpng.org/pub/png/spec/1.2/PNG-Structure.html 查看技术文档。 上面有PNG图像二进制文件的字节格式的详细信息，该文件由重复字节块组成。

在十六进制编辑器中打开PNG文件，浏览每个相关字节块组件查看其作用。 我们使用 Linux 上原生的 hexdump 十六进制编辑器，其实任何十六进制编辑器都应该可以。 可以在https://github.com/blackhat-go/bhg/blob/master/ch-13/imgInject/images/battlecat.png 上查看我们打开的示例图片；不过，所有有效的PNG图片都将遵循相同的格式。

### 头

图像文件的前8个字节，89 50 4e 47 0d 0a 1a 0a，被称为 `header` 在图13-1中高亮标记。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-13/images/13-1.png)
图13-1：PNG文件头

转换为ASCII时，逐个读取PNG的第二、三、四个十六进制值。任意的末尾字节由DOS和Unix Carriage-Return Line Feed (CRLF)组成。这种特定的头序列，称为文件的 `magic bytes`，在每个有效的PNG文件中都是相同的。内容的变化发生在剩余的块中，很快就会看到。

我们按照规范来完成，从在Go中构建表示PNG格式开始。 这将帮助我们加快嵌入有效载荷的最终目标。 因为头长是8字节，可以封装成uint64数据类型，因此继续构建 `Header` 结构体来保存该值（清单 13-1）。 （所有代码清单在github https://github.com/blackhat-go/bhg/ 跟目录下。)

```go
//Header holds the first UINT64 (Magic Bytes)
type Header struct {
	Header uint64
}
```
清单 13-1：定义Header结构体 (/ch-13/imgInject/pnglib/commands.go)

### 序列块

PNG文件的其余部分，如图13-2所示，由遵循此模式的重复字节块组成:SIZE(4字节)，TYPE(4字节)，DATA(任意数量的字节)，和CRC(4字节)。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-13/images/13-2.png)
图13-图像数据剩余块及模式

进一步查看十六进制转储，您可以看到第一个块——SIZE块——由字节0x00 0x00 0x00 0x0d组成。 这个块定义接下来的DATA块的长度。 该十六进制转换成ASCII是13——这表示DATA块由13个字节组成。 TYPE 块的字节是 0x49 0x48 0x44 0x52，本例中转换成IHDR的ASCII值。PNG规范定义了各种有效的类型。 其中一些类型，如IHDR用于定义图像的元数据，或标志图像数据流的结束。其他类型，特别是IDAT类型，表示实际图像的字节。

接下来是DATA块，长度由SIZE块定义。 最后，CRC块结束了整个块段。 它由 TYPE 和 DATA 字节的 CRC-32 校验和组成。 这个CRC块的字节是 0x9a 0x76 0x82 0x70。 这种格式在整个图像文件中不断重复，直到End of File (EOF)状态，该状态由类型为IEND的块表示。

像清单13-1中那样定义Header结构，构建一个结构来保存单个块，如清单13-2中所示。

```go
//Chunk represents a data byte chunk segment
type Chunk struct {
	Size uint32
	Type uint32
	Data []byte
	CRC uint32
}
```
清单 13-2：定义Chunk结构体 (/ch-13/imgInject/pnglib/commands.go)