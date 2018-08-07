## Lập trình Blockchain với Golang. Part 1: Cơ bản

>Bài dịch từ _Building Blockchain in Go_ của tác giả _Ivan Kuznetsov_. Khi sử dụng vui lòng trích dẫn nguồn [@hlongvu](http://github.com/hlongvu)

### Blockchain
Blockchain được xem là một trong những công nghệ mang tính cách mạng trong thế kỷ 21, đang dần trưởng thành và có những tiềm năng chưa khai phá hết. Về bản chất, blockchain chỉ là một bộ dữ liệu phân tán. Nhưng điều đặc biệt là nó không phải một cơ sở dữ liệu riêng tư (private database), mà nó là cơ sở dữ liệu mở (public database), bất kì ai cũng có thể sao chép tất cả (hoặc từng phần) bộ cơ sở dữ liệu này. Hơn nữa, nhờ có blockchain thì cryptocurrency (tiền-mã-hoá) và smart contracts mới có thể hiện thực hoá được.

Trong seri bài hướng dẫn này, chúng ta sẽ xây dựng một cryptocurrency đạng đơn giản, dựa trên công nghệ blockchain.

### Block

Trong blockchain, block chính là phần chứa những dữ liệu có giá trị. Ví dụ, các block của Bitcoin chứa các transactions (giao dịch), phần cốt yếu của bất kì cryptocurrency nào. Ngoài những giao dịch này, block còn chứa những thông tin kỹ thuật khác, ví dụ version, timestamp và hash (mã băm) của block trước.

Trong hướng dẫn này chúng ta sẽ không xây dựng đầy đủ theo đặc tả của Bitcoin, mà chúng ta sẽ làm theo một cách đơn giản hơn, chỉ chứa những thông tin quan trọng và cốt yếu để blockchain có thể hoạt động được. Như sau:

```
type Block struct {
	Timestamp     int64
	Data          []byte
	PrevBlockHash []byte
	Hash          []byte
}
```

**Timestamp** là thời điểm block được tạo, **Data** là những thông tin quan trọng mà block cần chứa, **PrevBlockHash** là hash của block trước đó, **Hash** chính là hash của block này.

Trong đặc tả kỹ thuật của Bitcoin, **Timestamp**,  **PrevBlockHash**, **Hash** nằm trong phần block headers, còn các transactions (như **Data** ở trên) được lưu riêng ở phần dữ liệu khác. Hiện tại chúng ta đang lưu các trường chung trong một struct để đơn giản hoá.

Làm cách nào để tính các hash? Thuật toán để tính ra hash (hàm băm) đóng một vai trò rất quan trọng và chính là để bảo vệ blockchain. Hàm băm phải đòi hỏi tiêu tốn nhiều công sức (computationally difficult operation), thậm chí phải đầu tư CPU và GPU, ASICS cho việc tính toán này. Đây không phải là ngẫu nhiên mà đã được thiết kế có chủ định (by design), nhằm ngăn chặn việc có thể tạo ra các block một cách dễ dàng, khiến blockchain có thể bị thay đổi bằng các dữ liệu giả mạo. Chúng ta sẽ thảo luận thêm về thuật toán băm này trong các phần sau.

Hiện tại chúng ta chỉ cần lấy hàm băm SHA-256 của phần dữ liệu trong Block. Các dữ liệu được chuyển sang byte và ghép nối với nhau, sau đó lấy mã băm sha256.

```
func (b *Block) SetHash() {
	timestamp := []byte(strconv.FormatInt(b.Timestamp, 10))
	headers := bytes.Join([][]byte{b.PrevBlockHash, b.Data, timestamp}, []byte{})
	hash := sha256.Sum256(headers)

	b.Hash = hash[:]
}
```

Và đây là hàm tạo một Block mới:

```
func NewBlock(data string, prevBlockHash []byte) *Block {
	block := &Block{time.Now().Unix(), []byte(data), prevBlockHash, []byte{}}
	block.SetHash()
	return block
}
```



### Blockchain

Về cơ bản blockchain là một bộ dữ liệu có cấu trúc như một linked-list. Các block được lưu theo thứ tự và mỗi block đều có con trỏ link tới block liền kề phía trước. Chúng ta có thể truy cập được ngay block cuối cùng, và lấy block theo mã hash của block đó.

Chúng ta có thể xây dựng cấu trúc này bằng một array và một map: array sẽ lưu hash của các block, map sẽ lưu (hash: block). Blockchain đơn giản của chúng ta đang chỉ cần lưu array các block. Như sau:

```
type Blockchain struct {
	blocks []*Block
}
```
Đây chính là phiên bản blockchain đầu tiên của chúng ta 😉.

Hãy tạo hàm để thêm block vào blockchain:

```
func (bc *Blockchain) AddBlock(data string) {
	prevBlock := bc.blocks[len(bc.blocks)-1]
	newBlock := NewBlock(data, prevBlock.Hash)
	bc.blocks = append(bc.blocks, newBlock)
}
```

Để thêm block mới vào blockchain chúng ta cần có block liền trước nó. Vậy lúc blockchain chưa có block nào thì sao? Trong bất kì blockchain nào cũng phải có block đầu tiên này, và nó được gọi là **genesis block**. Đây là hàm để khởi tạo genesis block này:

```
func NewGenesisBlock() *Block {
	return NewBlock("Genesis Block", []byte{})
}
```

Tiếp theo là hàm tạo blockchain với genesis block này:

```
func NewBlockchain() *Blockchain {
	return &Blockchain{[]*Block{NewGenesisBlock()}}
}
```


Như vậy là đã hình thành blockchain đơn giản: có genesis block và có thể thêm các block tiếp theo vào blockchain.
Hãy kiểm tra xem blockchain này đã hoạt động chưa:

```
func main() {
	bc := NewBlockchain()

	bc.AddBlock("Send 1 BTC to Ivan")
	bc.AddBlock("Send 2 more BTC to Ivan")

	for _, block := range bc.blocks {
		fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
		fmt.Printf("Data: %s\n", block.Data)
		fmt.Printf("Hash: %x\n", block.Hash)
		fmt.Println()
	}
}
```

Kết quả:

```
Prev. hash:
Data: Genesis Block
Hash: aff955a50dc6cd2abfe81b8849eab15f99ed1dc333d38487024223b5fe0f1168

Prev. hash: aff955a50dc6cd2abfe81b8849eab15f99ed1dc333d38487024223b5fe0f1168
Data: Send 1 BTC to Ivan
Hash: d75ce22a840abb9b4e8fc3b60767c4ba3f46a0432d3ea15b71aef9fde6a314e1

Prev. hash: d75ce22a840abb9b4e8fc3b60767c4ba3f46a0432d3ea15b71aef9fde6a314e1
Data: Send 2 more BTC to Ivan
Hash: 561237522bb7fcfbccbc6fe0e98bbbde7427ffe01c6fb223f7562288ca2295d1
```

### Kết luận
Chúng ta vừa xây dựng một blockchain đơn giản: chỉ bao gồm một array các block, mỗi block có prevBlockHash trỏ tới block liền kề trước nó. Trong thực tế blockchain sẽ phức tạp hơn nhiều. Blockchain đơn giản trên có thể thêm block vào tuỳ ý và rất nhanh bởi hàm AddBlock, nhưng một blockchain thực thụ cần phải qua các bước tính toán nặng hơn nhiều: mỗi máy tính phải xử lí một khối lượng công việc rất lớn và đáp ứng điều kiện mới được quyền thêm block vào blockchain (gọi là Proof-of-Work). Ngoài ra blockchain còn là một bộ dữ liệu phân tán, và không ai có quyền làm chủ. Do đó mỗi block mới muốn được thêm vào blockchain phải được tất cả các máy tính khác trong mạng cho phép bằng một luật chung (consensus).

Blockchain của chúng ta cũng chưa có một transaction nào cả. Trong phần sau chúng ta sẽ thêm các tính năng mới vào blockchain này.

### Links

1. Full source codes:  [https://github.com/Jeiwan/blockchain_go/tree/part_1](https://github.com/Jeiwan/blockchain_go/tree/part_1)
2. Block hashing algorithm: [https://en.bitcoin.it/wiki/Block_hashing_algorithm](https://en.bitcoin.it/wiki/Block_hashing_algorithm)




