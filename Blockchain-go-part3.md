## Lập trình Blockchain với Golang. Part 3: Lưu trữ và tương tác CLI

>Bài dịch từ _Building Blockchain in Go_ của tác giả _Ivan Kuznetsov_. Khi sử dụng vui lòng trích dẫn nguồn [@hlongvu](https://github.com/hlongvu/blockchain-go-vietnamese)


### Mục lục

1. [Lập trình Blockchain với Golang. Part 1: Cơ bản](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part1.md)
2. [Lập trình Blockchain với Golang. Part 2: Proof-of-work](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part2.md)
3. [Lập trình Blockchain với Golang. Part 3: Lưu trữ và tương tác CLI](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part3.md)
4. [Lập trình Blockchain với Golang. Part 4: Transactions 1](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part4.md)
5. [Lập trình Blockchain với Golang. Part 5: Address](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part5.md) 
6. [Lập trình Blockchain với Golang. Part 6: Transaction 2](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part6.md)
7. [Lập trình Blockchain với Golang. Part 7: Network](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part7.md)


### Giới thiệu
Phần này chúng ta sẽ tiến hành lưu trữ blockchain trong một database (DB) và xây dựng thêm command-line để tương tác với nó. Về cơ bản, blockchain là một bộ dữ liệu phân tán, tuy nhiên chúng ta sẽ lưu trữ dữ liệu trước và phần phân tán sẽ tiến hành sau.

### Lựa chọn database
Bạn có thể sử dụng loại database nào? Sự thật là bạn có thể sử dụng bất cứ loại nào. Trong Bitcoin whitepaper cũng không hề chỉ định một loại database nào cụ thể, tuỳ vào lập trình viên lựa chọn mà thôi. Phiên bản [Bitcoin Core](https://github.com/bitcoin/bitcoin) được Satoshi Nakamoto lập trình sử dụng [LevelDB](https://github.com/google/leveldb). Còn với chúng ta khi sử dụng ngôn ngữ Golang thì sẽ lựa chọn...

### BoltDB
Bởi:

1. Tính dễ dùng và tối giản
2. BoltDB được viết bởi Go
3. BoltDB không cần phải chạy trên một server
4. Các cấu trúc dữ liệu chúng ta muốn đều có thể sử dụng BoltDB

Từ giới thiệu của BoltDB trên Github:
> BoltDB là hệ lưu trữ key/value viết bằng Golang và được truyền cảm hứng bởi LMDB. Mục tiêu của nó là cung cấp một cơ sở dữ liệu đơn giản, nhanh và tin cậy cho những project không cần tới hệ quản trị cơ sở dữ liệu đầy đủ như Postgres hoặc MySQL
> 
> BoltDB được dùng ở những tác vụ cấp thấp nên tính đơn giản được đề cao trên hết. API của nó rất gọn nhẹ và chỉ tập trung vào việc lấy và lưu dữ liệu.

Xem chừng chỉ đó là đã đủ để chúng ta sử dụng trong project này. Hãy review lại đôi chút:

BoltDB là một bộ lưu dữ liệu theo key/value, có nghĩa là không cần tới table như ở SQL RDBMS (MySQL, PostgreSQL,...), không hàng, cột. Thay vào đó dữ liệu được lưu theo key-value (giống Golang map). Các cặp key-value này được lưu trong các bucket, mục đích là để gộp các dữ liệu giống nhau lại. Vậy, để lấy ra một giá trị (value), chúng ta cần biết bucket và key.

Một chú ý quan trọng trong BoltDB là nó không có kiểu dữ liệu: key và value đều ở dạng mảng byte. Để có thể lưu trữ Go struct, chúng ta phải serialize  chúng - chuyển struct thành byte array và chuyển ngược lại byte array thành struct. Chúng ta sẽ sử dụng thư viện [encoding/gob](https://golang.org/pkg/encoding/gob/) để làm điều này. Mặc dù có thể dùng các kiểu dữ liệu khác JSON, XML, Protocol Bufers,... nhưng chúng ta sử dụng encoding/gob vì tính đơn giản và được hỗ trợ trực tiếp từ thư viện chuẩn của Go.

### Cấu trúc dữ liệu

Trước khi tiến hành code phần lưu trữ dữ liệu, chúng ta hãy nghiên cứu xem nên lưu dữ liệu như thế nào. Hãy tham khảo xem Bitcoin Core thực hiện ra sao.

Bitcoin Core sử dụng 2 "buckets" để lưu dữ liệu:

1. blocks để lưu các dữ liệu về block trong chuỗi
2. chainstate để lưu trạng thái của chuỗi, ví dụ các giao dịch chưa sử dụng (unspent transaction outputs) và một số dữ liệu kỹ thuật khác

Thêm vào đó, blocks được lưu ra nhiều file trên ổ đĩa. Việc này là để tối ưu tốc độ đọc ghi. Chúng ta chưa cần thiết phải xây dựng phần này.

Trong **blocks**, các cặp **key->value** là:

1. **'b' + 32-byte block hash -> block index record**
2. **'f' + 4-byte file number -> file information record**
3. **'l' -> 4-byte file number: the last block file number used**
4. **'R' -> 1-byte boolean: whether we're in the process of reindexing**
5. **'F' + 1-byte flag name length + flag name string -> 1 byte boolean: various flags that can be on or off**
6. **'t' + 32-byte transaction hash -> transaction index record**

Trong **chainstats**, các cặp **key->value** là:

1. **'c' + 32-byte transaction hash -> unspent transaction output record for that transaction**
2. **'B' -> 32-byte block hash: the block hash up to which the database represents the unspent transaction outputs**

(Đặc tả cụ thể các bạn có thể xem [tại đây](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage))

Trong blockchain của chúng ta chưa có transaction nào cả, vì thế chúng ta chỉ sử dụng **blocks** bucket, và bucket này cũng chỉ lưu trong 1 file. Vậy các cặp **key->value** mà chúng ta có là:

1. **'b' + 32-byte block hash -> block index record**
3. **'l' -> hash của block cuối cùng trong chuỗi**

Vậy hãy bắt tay vào code.

### Serialization

Như đã nói, boltDB chỉ có thể lưu trữ kiểu []byte trong khi đó chúng ta lại lưu Block kiểu struct. Sử dụng encoding/gob để serialize:

```
func (b *Block) Serialize() []byte {
	var result bytes.Buffer
	encoder := gob.NewEncoder(&result)

	err := encoder.Encode(b)

	return result.Bytes()
}
```
Tiếp theo là hàm Deserialize:

```
func DeserializeBlock(d []byte) *Block {
	var block Block

	decoder := gob.NewDecoder(bytes.NewReader(d))
	err := decoder.Decode(&block)

	return &block
}
```

>Các bạn có thể tham khảo thêm thư viện [encoding/gob](https://golang.org/pkg/encoding/gob/).

### Lưu trữ

Bắt đầu với hàm **NewBlockchain**, hiện tại hàm này tạo ra một **Blockchain** và thêm genesis block vào đó. Chúng ra sẽ sửa lại như sau:

1. Mở một file DB
2. Kiểm tra xem đã có blockchain nào trong dữ liệu chưa
3. Nếu đã có:
	- Tạo một instance Blockchain mới
	- Đặt đỉnh (tip) của Blockchain này là hash cuối đã lưu trong DB
4. Nếu chưa có blockchain nào:
	- Tạo genesis block
	- Lưu lại trong DB
	- Lưu lại hash của genesis block vào hash cuối trong DB
	- Tạo một instance Blockchain với tip là hash đó	
Xây dựng bằng code sẽ như sau:

```
func NewBlockchain() *Blockchain {
	var tip []byte
	db, err := bolt.Open(dbFile, 0600, nil)

	err = db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))

		if b == nil {
			genesis := NewGenesisBlock()
			b, err := tx.CreateBucket([]byte(blocksBucket))
			err = b.Put(genesis.Hash, genesis.Serialize())
			err = b.Put([]byte("l"), genesis.Hash)
			tip = genesis.Hash
		} else {
			tip = b.Get([]byte("l"))
		}

		return nil
	})

	bc := Blockchain{tip, db}

	return &bc
}
```

Hãy xem lại các bước:

```
db, err := bolt.Open(dbFile, 0600, nil)
```
Đây là câu lệnh để mở một file BoltDB. Chú ý là nó sẽ không trả về error nếu file không tồn tại.

```
err = db.Update(func(tx *bolt.Tx) error {
...
})
```
Trong BoltDB, mỗi thao tác với dữ liệu được chạy trong một transaction (Giao dịch). Có 2 loại transaction: chỉ đọc và đọc-ghi. Ở đây chúng ta mở một transaction đọc ghi ( **db.Update(...)** ) để có thể ghi genesis block vào DB.

```
b := tx.Bucket([]byte(blocksBucket))

if b == nil {
	genesis := NewGenesisBlock()
	b, err := tx.CreateBucket([]byte(blocksBucket))
	err = b.Put(genesis.Hash, genesis.Serialize())
	err = b.Put([]byte("l"), genesis.Hash)
	tip = genesis.Hash
} else {
	tip = b.Get([]byte("l"))
}
```
Đây là phần chính của hàm NewBlockchain. Ở đây chúng ta kiểm tra xem blockchain đã tồn tại chưa. Nếu chưa có, chúng ta tạo ra genesis block, tạo bucket và lưu block và cập nhật key "l" lưu lại hash của block cuối trong chuỗi.

Đồng thời chúng ta cũng tạo ra instance Blockchain:

```
bc := Blockchain{tip, db}
```

Chúng ta không lưu blockchain bằng mảng block nữa, chúng ta chỉ lưu lại đỉnh (tip), là hash của block cuối cùng. Chúng ta cũng giữ lại một biến db để có thể truy cập nhanh tới blockchain. Struct mới của chúng ta sẽ như sau:

```
type Blockchain struct {
	tip []byte
	db  *bolt.DB
}
```

Tiếp theo chúng ta sẽ cập nhật lại hàm **AddBlock**, không đơn giản như việc thêm một block mới vào mảng nữa, chúng ta phải lưu vào DB.

```
func (bc *Blockchain) AddBlock(data string) {
	var lastHash []byte

	err := bc.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		lastHash = b.Get([]byte("l"))

		return nil
	})

	newBlock := NewBlock(data, lastHash)

	err = bc.db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		err := b.Put(newBlock.Hash, newBlock.Serialize())
		err = b.Put([]byte("l"), newBlock.Hash)
		bc.tip = newBlock.Hash

		return nil
	})
}
```
Hãy xem lại từng bước:
```
err := bc.db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte(blocksBucket))
	lastHash = b.Get([]byte("l"))

	return nil
})
```
Phía trên là 1 transaction read-only để lấy hash cuối cùng của blockchain. Hash này sẽ được dùng để mine block tiếp theo.

```
newBlock := NewBlock(data, lastHash)
b := tx.Bucket([]byte(blocksBucket))
err := b.Put(newBlock.Hash, newBlock.Serialize())
err = b.Put([]byte("l"), newBlock.Hash)
bc.tip = newBlock.Hash
```
Sau khi mine được newBlock chúng ta lưu lại vào DB và cập nhật key "l" là hash của block mới này.

Dễ phải không?

### Kiểm tra Blockchain mới

Blockchain của chúng ta đã được lưu vào cơ sở dữ liệu, chúng ta có thể tắt mở chương trình và thêm block vào đó. Nhưng các block không lưu trong array nữa nên phần in ra các block đã không còn hoạt động. Hãy sửa phần này!

BoltDB cho phép chúng ta quét qua tất cả các key trong bucket, tuy nhiên các key lại sắp xếp theo thứ tự byte chứ không phải thứ tự block thêm vào. Đồng thời chúng ta cũng không muốn tải toàn bộ block vào bộ nhớ, vì vậy hãy đọc chúng từng block một. Để xây dựng tính năng này, chúng ta cần một bộ quét (iterator):

```
type BlockchainIterator struct {
	currentHash []byte
	db          *bolt.DB
}
```
Mỗi iterator sẽ được tạo lúc cần quét qua các blocks, iterator này sẽ lưu kết nối db và hash của block hiện tại đang trỏ tới. Iterator này được tạo ra từ Blockchain:

```
func (bc *Blockchain) Iterator() *BlockchainIterator {
	bci := &BlockchainIterator{bc.tip, bc.db}

	return bci
}
```
Chú ý là iterator này lúc khởi tạo cũng trỏ tới tip của blockchain, vì thế khi quét, các block sẽ lần lượt từ mới nhất tới cũ nhất.

Trong thực tế một blockchain có thể có nhiều nhánh, nhánh dài nhất sẽ được chọn làm nhánh chính. Tip này cũng có thể là bất kì block nào. Nên tip này có thể được hiểu như là một id của blockchain. Việc chọn một tip cũng là "voting" cho một blockchain. 

**BlockchainIterator** chỉ thực hiện một nhiệm vụ: trả về block tiếp theo trong blockchain:

```
func (i *BlockchainIterator) Next() *Block {
	var block *Block

	err := i.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		encodedBlock := b.Get(i.currentHash)
		block = DeserializeBlock(encodedBlock)

		return nil
	})

	i.currentHash = block.PrevBlockHash

	return block
}
```

Phần DB đến đây cơ bản đã xong.

### CLI

Blockchain hiện tại chỉ mới chạy trong hàm main mà chưa cho phép tương tác nào. Chúng ta sẽ tiếp tục xây dựng các command cho chương trình như sau:

```
blockchain_go addblock "Pay 0.031337 for a coffee"
blockchain_go printchain
```

Các command này sẽ được thực thi qua CLI struct:

```
type CLI struct {
	bc *Blockchain
}
```

Hàm **Run** sẽ chạy các command:

```
func (cli *CLI) Run() {
	cli.validateArgs()

	addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
	printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)

	addBlockData := addBlockCmd.String("data", "", "Block data")

	switch os.Args[1] {
	case "addblock":
		err := addBlockCmd.Parse(os.Args[2:])
	case "printchain":
		err := printChainCmd.Parse(os.Args[2:])
	default:
		cli.printUsage()
		os.Exit(1)
	}

	if addBlockCmd.Parsed() {
		if *addBlockData == "" {
			addBlockCmd.Usage()
			os.Exit(1)
		}
		cli.addBlock(*addBlockData)
	}

	if printChainCmd.Parsed() {
		cli.printChain()
	}
}
```

Các command được thư viện [flag](https://golang.org/pkg/flag/) parse.

```
addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
addBlockData := addBlockCmd.String("data", "", "Block data")
```
Hàm addblock sẽ có thêm trường data là dữ liệu muốn lưu.

```
switch os.Args[1] {
case "addblock":
	err := addBlockCmd.Parse(os.Args[2:])
case "printchain":
	err := printChainCmd.Parse(os.Args[2:])
default:
	cli.printUsage()
	os.Exit(1)
}
```

Tiếp theo chúng ta sẽ kiểm tra command và trường đi kèm.

```
if addBlockCmd.Parsed() {
	if *addBlockData == "" {
		addBlockCmd.Usage()
		os.Exit(1)
	}
	cli.addBlock(*addBlockData)
}

if printChainCmd.Parsed() {
	cli.printChain()
}
```
Tiếp theo là check và chạy hàm tương ứng.

```
func (cli *CLI) addBlock(data string) {
	cli.bc.AddBlock(data)
	fmt.Println("Success!")
}

func (cli *CLI) printChain() {
	bci := cli.bc.Iterator()

	for {
		block := bci.Next()

		fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
		fmt.Printf("Data: %s\n", block.Data)
		fmt.Printf("Hash: %x\n", block.Hash)
		pow := NewProofOfWork(block)
		fmt.Printf("PoW: %s\n", strconv.FormatBool(pow.Validate()))
		fmt.Println()

		if len(block.PrevBlockHash) == 0 {
			break
		}
	}
}
```

Các hàm trên tương tự như chúng ta đã xây dựng, chỉ có bây giờ chúng ta dùng BlockchainIterator để quét qua blockchain.

Hàm main của chúng ta giờ được cập nhật lại như sau:

```
func main() {
	bc := NewBlockchain()
	defer bc.db.Close()

	cli := CLI{bc}
	cli.Run()
}
```

Chú ý: Dù bạn chạy command nào thì một Blockchain cũng được tạo ra.

Bây giờ hãy kiểm tra xem chương trình có chạy đúng không:

```
$ blockchain_go printchain
No existing blockchain found. Creating a new one...
Mining the block containing "Genesis Block"
000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true

$ blockchain_go addblock -data "Send 1 BTC to Ivan"
Mining the block containing "Send 1 BTC to Ivan"
000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13

Success!

$ blockchain_go addblock -data "Pay 0.31337 BTC for a coffee"
Mining the block containing "Pay 0.31337 BTC for a coffee"
000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148

Success!

$ blockchain_go printchain
Prev. hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
Data: Pay 0.31337 BTC for a coffee
Hash: 000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148
PoW: true

Prev. hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
Data: Send 1 BTC to Ivan
Hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
PoW: true

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true
```

### Kết luận

Ở phần sau chúng ta sẽ xây dựng address, wallet và transactions.

### Links

1. [Full source codes](https://github.com/Jeiwan/blockchain_go/tree/part_3)
2. [Bitcoin Core Data Storage](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage)
3. [boltdb](https://github.com/boltdb/bolt)
4. [encoding/gob](https://golang.org/pkg/encoding/gob/)
5. [flag](https://golang.org/pkg/flag/)





### Mục lục

1. [Lập trình Blockchain với Golang. Part 1: Cơ bản](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part1.md)
2. [Lập trình Blockchain với Golang. Part 2: Proof-of-work](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part2.md)
3. [Lập trình Blockchain với Golang. Part 3: Lưu trữ và tương tác CLI](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part3.md)
4. [Lập trình Blockchain với Golang. Part 4: Transactions 1](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part4.md)
5. [Lập trình Blockchain với Golang. Part 5: Address](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part5.md) 
6. [Lập trình Blockchain với Golang. Part 6: Transaction 2](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part6.md)
7. [Lập trình Blockchain với Golang. Part 7: Network](https://github.com/hlongvu/blockchain-go-vietnamese/blob/master/Blockchain-go-part7.md)
