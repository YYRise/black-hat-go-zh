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
图13-2 图像数据剩余块及模式

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


## 读取图片的字节数据

Go语言能相对容易地处理二进制数据的读写，这在一定程度上要归功于 `binary` 包（在第6章的时候介绍过），但是在解析PNG数据前需要先打开文件来读取。 创建`PreProcessImage()` 函数，参数为 `*os.File` 类型的文件句柄，且返回值为 `*bytes.Reader` 类型（清单13-3）。

```go
//PreProcessImage reads to buffer from file handle
func PreProcessImage(dat *os.File) (*bytes.Reader, error) {
	stats, err := dat.Stat()
	if err != nil {
		return nil, err
	}
	var size = stats.Size()
	b := make([]byte, size)
	bufR := bufio.NewReader(dat)
	_, err = bufR.Read(b)
	bReader := bytes.NewReader(b)

	return bReader, err
}
```
清单 13-3：定义 `PreProcessImage()` 函数 (/ch-13/imgInject/utils/reader.go)

函数打开文件对象，以获取用于抓取大小信息的 `FileInfo` 结构体。 接下来的几行代码用过 `bufio.NewReader()` 实例化一个 `Reader` 实例，然后调用 `bytes.NewReader()` 生成 `*bytes.Reader` 实例。 函数返回一个 `*bytes.Reader` 类型，用于开始使用 `binary` 包读取字节数据。 先读取头数据，再读取序列块。

### 读取头数据

使用定义PNG文件的前8个字节验证文件是否是PNG文件，构建 `validate()` 方法（清单13-4）。

```go
func (mc *MetaChunk) validate(b *bytes.Reader) {
	var header Header

	if err := binary.Read(b, binary.BigEndian, &header.Header); err != nil {
		log.Fatal(err)
	}
	bArr := make([]byte, 8)
	binary.BigEndian.PutUint64(bArr, header.Header)
	if string(bArr[1:4]) != "PNG" {
		log.Fatal("Provided file is not a valid PNG format")
	} else {
		fmt.Println("Valid PNG so let us continue!")
	}
}
```
清单 13-4: 验证是否是 PNG 文件 (/ch-13/imgInject/pnglib/commands.go)

尽管此方法可能看起来并不太复杂，但它引入了几个新条目。 首先，也是最明显的一个是 `binary.Read()` 函数，该函数从 `bytes.Reader` 复制8个字节到 `Header` 结构体中。 还记得定义的定义的 `Header` 结构体中，字段类型为 `uint64`（清单13-1），相当于8个字节。 同样值得注意的是， `binary` 包提供了通过 `binary.BigEndian` 和 `binary.LittleEndian` 分别读取 `Most Significant Bit` 和 `Least Significant Bit` 格式的方法。 当进行二进制写时，这些函数也十分有用；例如，可以使用 `BigEndian` 将字节放在指定使用网络字节排序的线路上。

二进制字节序函数还包含有助于将数据类型编码为文字数据类型（例如 uint64）的方法。 这里创建了一个长度为8的字节数组，并执行二进制读取，将数据复制到`uint64`数据类型中。 然后将字节转换为字符串形式，并使用切片和简单的字符串比较来确认1到4字节是PNG字符串，标志着这是个有效的图片文件格式。

为优化查验文件是否为PNG文件这一过程，我们鼓励你查阅Go的`bytes` 包，因为含有简便的函数，可以使用这些函数快捷地进行文件头和之前提到过的PNG魔法字节序列的比较。 自己去研究吧。

### 读取序列块

一旦验证了是PNG文件，就可以写读取序列块的代码了。 头在PNG文件中只出现一次，但序列块是`SIZE, TYPE, DATA, 和 CRC` 的重复，直到`EOF`。 因此，需要合适地处理这种重复，Go中的条件循环可以方便地做到。 基于此，构建 `ProcessImage()` 方法，迭代地处理所有的数据块，直到文件末尾（清单13-5）。

```go
func (mc *MetaChunk) ProcessImage(b *bytes.Reader, c *models.CmdLineOpts) {
// Snip code for brevity (Only displaying relevant lines from code block)
	count := 1 //Start at 1 because 0 is reserved for magic byte
	chunkType := ""
	endChunkType := "IEND" //The last TYPE prior to EOF
	for chunkType != endChunkType {
		fmt.Println("---- Chunk # " + strconv.Itoa(count) + " ----")
		offset := chk.getOffset(b)
		fmt.Printf("Chunk Offset: %#02x\n", offset)
		chk.readChunk(b)
		chunkType = chk.chunkTypeToString()
		count++
	}
}
```
清单13-5：`ProcessImage()` 方法 (/ch-13/imgInject/pnglib/commands.go)

首先传递将一个对 `bytes.Reader` 内存地址指针 (`*bytes.Reader`) 的引用作为参数传递给 `ProcessImage()`。 刚刚创建的`validate()` 方法（清单13-4）也使用一个 `bytes.Reader` 指针的引用。 按照约定，对同一个内存地址指针位置的多次引用将允许对引用数据的可变访问。 这实质上意味着将 `bytes.Reader` 引用作为参数传递给 `ProcessImage()` 时，由于Header的大小，读取器已经前进了8个字节，因为访问的是相同的 `bytes.Reader` 实例。

或者，没有传递指针， `bytes.Reader` 要么是相同的PNG图像数据的副本，要么是单独的唯一实例数据。 这是因为，读取header时推进指针不会在其他地方适当地推进读取器。 需要避免采用这种方式。 一方面，在不必要的情况下传递多个数据副本只是一种糟糕的约定。 更重要的是，每次传递副本时，它都被定位在文件的开头，迫使在读取块序列之前通过编程定义和管理它在文件中的位置。

随着代码块的进行，定义 `count` 变量来记录图像文件包含多少块段。 `chunkType` 和 `endChunkType` 作为比较逻辑的一部分，将 `chunkType` 和 `endChunkType’s IEND` 来比较作为 EOF 的条件。

最好知道每个块段从哪里开始——或者更确切地说，每个块在文件字节结构中的绝对位置，被称为 `offset`。 如果知道偏移值，非常容易地将负载植入到文件中。 例如，可以将一组偏移位置提供给解码器—— 一个单独的函数，用于收集每个已知偏移处的字节，然后将它们展开为您想要的有效负载。 要取到每个块的偏移，需要调用 `mc.getOffset(b)` 方法（清单13-6）。

```go
func (mc *MetaChunk) getOffset(b *bytes.Reader) {
	offset, _ := b.Seek(0, 1)
	mc.Offset = offset
}
```
清单13-6：`getOffset(b)` 方法 (/ch-13/imgInject/pnglib/commands.go)

`bytes.Reader` 中有个 `Seek()` 方法，该方法可以非常简单地取得当前位置。 `Seek()` 方法移动当前的读取或写入偏移，然后返回相对于文件开头的新的偏移。 该方法的第一个参数是要移动偏移量的字节数，第二个参数定义将要发生移动的位置。 第二个参数的可选值是0（文件开头），1（当前位置），和2（文件末尾）。 例如，想从当前位置左移8字节，可以使用 `b.Seek(-8,1)`。

在这里，`b.Seek(0,1)` 表示从当前位置偏移0字节，因此仅仅返回当前偏移：本质上检索偏移量而不移动它。

下面的方法中详细定义了如何读取实际的块段字节。 为了更容易理解，创建`readChunk()` 方法，然后创建读取每个子块的单独方法（清单13-7）。

```go
func (mc *MetaChunk) readChunk(b *bytes.Reader) {
	mc.readChunkSize(b)
	mc.readChunkType(b)
	mc.readChunkBytes(b, mc.Chk.Size) 
	mc.readChunkCRC(b)
}
func (mc *MetaChunk) readChunkSize(b *bytes.Reader) {
	if err := binary.Read(b, binary.BigEndian, &mc.Chk.Size); err != nil { 
		log.Fatal(err)
	}
}
func (mc *MetaChunk) readChunkType(b *bytes.Reader) {
	if err := binary.Read(b, binary.BigEndian, &mc.Chk.Type); err != nil {
		log.Fatal(err)
	}
}
func (mc *MetaChunk) readChunkBytes(b *bytes.Reader, cLen uint32) {
	mc.Chk.Data = make([]byte, cLen) 
	if err := binary.Read(b, binary.BigEndian, &mc.Chk.Data); err != nil {
		log.Fatal(err)
	}
}
func (mc *MetaChunk) readChunkCRC(b *bytes.Reader) {
	if err := binary.Read(b, binary.BigEndian, &mc.Chk.CRC); err != nil {
		log.Fatal(err)
	}
}
```
清单 13-7：块读取方法 (/ch-13/imgInject/pnglib /commands.go)

`readChunkSize(), readChunkType(), 和 readChunkCRC()` 是相似的。 每个读取 `uint32` 值到各自的 `Chunk` 结构体的字段中。 然而，`readChunkBytes()` 有点不一样。 因为图片数据的长度是可变的，需要将这个长度提供给`readChunkBytes()` 函数，以便知道要读取多少字节。 回想下，数据长度是在`SIZE`子块中维护的。 识别出 `SIZE` 的值，将其作为参数传递给 `readChunkBytes()` 用于定义切片的合适大小。 只有这样，才能将字节数据读入结构体的 `data` 字段。 这就是读取数据的方法，让我们继续研究如何写入字节数据。

## 写入图像字节数据以植入有效载荷

尽管可以从许多复杂的隐写技术中进行选择来植入有效载荷，但在本节中，将重点介绍一种写入特定字节偏移量的方法。PNG 文件格式定义了规范中的 `critical` 和 `ancillary` 。critical 块是图像解码器处理图像所必需的。`ancillary` 是可选的，并提供对编码或解码不重要的各种元数据，例如时间戳和文本。

因此， `ancillary` 块类型提供了覆盖现有块或插入新块的理想位置。在这里，我们将展示如何将新的字节片插入到 `ancillary` 块段中。

### 定位块偏移量

首先，需要在`ancillary`数据中确定适当的偏移量。您可以发现`ancillary`块，因为它们总是以小写字母开头。再次使用十六进制编辑器打开原始 PNG 文件，并前进到十六进制转储的末尾。

每个有效的 PNG 图像都有一个`IEND` 块类型，指示文件的最后一个块（EOF 块）。 移动到最终 `SIZE` 块之前的4个字节，处于 `IEND` 块的起始偏移量和整个PNG文件中包含的任意(critical 或 ancillary)块的最后一个偏移量。 回想下辅助块是可选的，因此在本文中所检查的文件可能没有相同的辅助块，或者与此相关的任何辅助块。 在我们的示例中，`IEND`块的偏移量从字节偏移量`0x85258`开始(图13-3)。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-13/images/13-3.png)
图13-3 识别相对于 `IEND` 位置的块偏移

### 使用 `ProcessImage()` 方法写入字节

将有序字节写入字节流的标准方法是使用Go结构体。 让我们重新查看清单13-5中开始构建的 `ProcessImage()` 方法的另一部分，并浏览其中的细节。 清单13-8中的代码调用各个函数，将在本节中逐步构建这些函数。

```go
func (mc *MetaChunk) ProcessImage(b *bytes.Reader, c *models.CmdLineOpts)  {
	--snip--
	var m MetaChunk
	m.Chk.Data = []byte(c.Payload)
	m.Chk.Type = m.strToInt(c.Type)
	m.Chk.Size = m.createChunkSize()
	m.Chk.CRC = m.createChunkCRC()
	bm := m.marshalData(){
		bmb := bm.Bytes()
		fmt.Printf("Payload Original: % X\n", []byte(c.Payload))
		fmt.Printf("Payload: % X\n", m.Chk.Data)
	utils.WriteData(b, c, bmb)
}
```
清单13-8：
使用 `ProcessImage()` 方法写入字节(/ch-13/imgInject/pnglib/commands.go)

该方法使用 `byte.Reader` 和另一个结构体 `models.CmdLineOpts` 作为参数。 `CmdLineOpts` 结构体如清单13-9 所示，含有通过命令行传入的标志。 这些标志确定使用什么样的负载，及将其插入的图片数据中的位置。 因为你要写的字节遵循相同的结构化格式，从已经存在的块段读取，可以创建一个新的 `MetaChunk` 结构体实例来接受新的块段值。

下一步是将负载读取到字节切片中。 但是，需要额外的功能强制把文字标志放到可用的字节数组中。 我们来深入研究 `strToInt(), createChunkSize(), createChunkCRC(), MarshalData(), WriteData()` 这些方法的细节。

```go
package models

//CmdLineOpts represents the cli arguments
type CmdLineOpts struct {
	Input string
	Output string
	Meta bool
	Suppress bool
	Offset string
	Inject bool
	Payload string
	Type string
	Encode bool
	Decode bool
	Key string
}
```
清单 13-9：`CmdLineOpts` 结构体 (/ch-13/imgInject/models/opts.go)

#### `strToInt()` 方法

先从 `strToInt()` 方法开始（清单13-10）。

```go
func (mc *MetaChunk) strToInt(s string) uint32 {
	t := []byte(s)
	return binary.BigEndian.Uint32(t)
}
```
清单 13-10: `strToInt()` 方法(/ch-13/imgInject/pnglib/commands.go)

`strToInt()` 方法将 `string` 转换成 `uint32`，是 `Chunk` 结构体 `TYPE` 值所需要的数据类型。

#### `createChunkSize()` 方法

接下来使用 `createChunkSize()` 方法创建 `Chunk` 结构体 `SIZE` 值 （清单13-11）。

```go
func (mc *MetaChunk) createChunkSize() uint32 {
	return uint32(len(mc.Chk.Data))
}
```
清单 13-11: `createChunkSize()` 方法 (/ch-13/imgInject/pnglib/commands.go)

这个方法取得 `chk.DATA` 字节数组的长度，并将类型转换成 `uint32`。

#### `createChunkCRC()` 方法

回想一下，每个块段的CRC校验和包括 `TYPE` 和 `DATA` 字节。 用`createChunkCRC()` 方法计算该校验和。 该方法利用 Go 的 `hash/crc32` 包（清单13-12）。

```go
func (mc *MetaChunk) createChunkCRC() uint32 {
	bytesMSB := new(bytes.Buffer)
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.Type); err != nil { 
		log.Fatal(err)
	}
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.Data); err != nil { 
		log.Fatal(err)
	}
	return crc32.ChecksumIEEE(bytesMSB.Bytes()) 
}
```
清单 13-12:`createChunkCRC()` 方法 (/ch-13/imgInject/pnglib/commands.go)

在返回语句之前，声明 `bytes.Buffer`，并将 `TYPE` 和 `DATA` 字节写入其中。 缓冲区中的字节切片作为参数传递给 `ChecksumIEEE`, 且 `CRC-32` 校验和作为 `uint32` 字节类型被返回。 `return` 语句在这里做了所有繁重的工作，实际上是在必要的字节上计算校验和。

#### `marshalData()` 方法

块的所有必要部分都被分配给它们各自的结构体字段中，现在可以编码到 `bytes.Buffer` 中。 缓冲区提供要插入到新图像文件中的自定义块的原始字节。 清单 13-13 是 `marshalData()` 方法。

```go
func (mc *MetaChunk) marshalData() *bytes.Buffer {
	bytesMSB := new(bytes.Buffer) 
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.Size); err != nil { 
		log.Fatal(err)
	}
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.Type); err != nil { 
		log.Fatal(err)
	}
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.Data); err != nil { 
		log.Fatal(err)
	}
	if err := binary.Write(bytesMSB, binary.BigEndian, mc.Chk.CRC); err != nil { 
		log.Fatal(err)
	}
	return bytesMSB
}
```
清单 13-13: `marshalData()` 方法 (/ch-13/imgInject/pnglib/commands.go)

`marshalData()` 方法声明 `bytes.Buffer`，并将包括大小、类型、数据、校验和在内的块信息写入其中。 该方法将所有块段数据返回到单个合并的 `bytes.Buffer` 中。

#### `WriteData()` 函数

现在剩下要做的就是将新的块段字节写入到原始PNG图像文件的偏移量中。让我们瞥一眼 `WriteData()` 函数，它存在于我们创建的名为 `utils` 包中(清单13-14)。

```go
//WriteData writes new Chunk data to offset
func WriteData(r *bytes.Readeru, c *models.CmdLineOptsv, b []bytew) {
	offset, _ := strconv.ParseInt(c.Offset, 10, 64)
	w, err := os.Create(c.Output)
	if err != nil {
		log.Fatal("Fatal: Problem writing to the output file!")
	}
	defer w.Close()
	r.Seek(0, 0)
	var buff = make([]byte, offset)
	r.Read(buff)
	w.Write(buff)
	w.Write(b)
	
	_, err = io.Copy(w, r)
	if err == nil {
		fmt.Printf("Success: %s created\n", c.Output)
	}
}

```

清单 13-14: `WriteData()` 函数 (/ch-13/imgInject/utils/writer.go)

`WriteData()` 函数使用一个含有原始图片文件字节数据的 `bytes.Reader`，一个含有命令行参数值的 `models.CmdLineOpts` 结构体，和一个 保存有新块字节段的字节切片。 代码块以 `string-to-int64` 转换开始，以便从 `models.CmdLineOpts` 结构体中获取偏移值；这将帮助您将新的块段写入特定位置，而不会损坏其他块。 然后创建文件句柄，以便将新修改的PNG图片写入到磁盘。

调用 `r.Seek(0,0)` 函数回到 `bytes.Reader` 绝对位置的开头。回想下前8个字节是为PNG头预留的，所以新的输出PNG图像也要包含这些头字节是很重要的。实例化一个由 `offset` 确定长度的字节切片，将这些字节保存到切片中。然后从原始图片中读取一定数量的字节，并将相同的字节写入到新的图片文件中。现在原始和新图片中是一样的头了。

然后将新断块写入到新图片文件中。向新图片文件追加剩余的 `bytes.Reader` 字节（也就是原始图像的块段字节）。回想下 `bytes.Reader` 推进到的偏移位置，因为之前读取到的字节切片包含了从 `offset` 到EOF的字节。得到了一个新图片文件。新文件具有与原始图像相同的前导和尾块，但是它还包含作为新的辅助块注入的有效负载。

为了解迄今为止所做的项目进度，请参考 https://github.com/blackhat-go/bhg/tree/master/ch-13/imgInject/ 上的整个项目代码。`imgInject` 程序使用命令行参数，其中包含原始PNG图像文件、偏移位置、任意数据有效负载、自声明的任意块类型，以及修改后的PNG图像文件的输出文件名，如清单13-15所示。

```sh
$ go run main.go -i images/battlecat.png -o newPNGfile --inject –offset \ 0x85258 --payload 1234243525522552522452355525
```

清单 13-15: 运行 *imgInject* 命令行程序

如果一切都按照计划进行，`offset 0x85258` 现在应该包含一个新的rNDm块段，如图13-4所示。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-13/images/13-4.png)

图13-3 作为辅助块注入的有效载荷（例如*rNDm*）

恭喜——已经写了第一个隐写程序！