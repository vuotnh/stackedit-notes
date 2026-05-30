# Benchmarks trong Go
Function benchmark có dạng
```go
func BenchmarkXxx(b *testing.b)
```
- Các func này sẽ được thực thi khi chạy lệnh `go test -bench=.`. Tham khảo thêm trong [Testing flags](https://pkg.go.dev/cmd/go#hdr-Testing_flags)
- Func ví dụ
```go
func BenchmarkRandInt(b *testing.B) {
		//... setup ...
    for b.Loop() {
		    //... code to measure ...
        rand.Int()
    }
    //... cleanup ...
}
```
Chạy lệnh
```bash
# chạy tất cả các file test, trong đó chạy cả benchmark
go test .\... -bench ".*"
```
Ta có output
```bash
goos: windows
goarch: amd64
pkg: golearn/iteration
cpu: Intel(R) Core(TM) Ultra 7 155H
BenchmarkRepeat-22      15106842                69.91 ns/op
```
- Giải thích:
	- `15106842`: đoạn code trong loop của func benchmark đã chạy 15106842 lần
	- Thời gian chạy của mỗi lần là `69.91 ns per loop`.  ***Chú ý: đây là thời gian chạy của code to  measure, ko tính thời gian setup và cleanup***


# Strings
`Strings` trong Go là ***immutable***. 
The byte sequence contained in a string value can never be changed.
Ví dụ:
```go
s := "left foot"  
t := s
s += ", right foot"
```
1. Lúc đầu, cả 2 biến `s` và `t` cùng trỏ tới một ô dữ liêu string (value semantics)
```bash
s ----\  
       -> "hello"  
t ----/
```
2. Khi nối 2 string `s +=  ", right foot"`, Go ***KO*** sửa string cũ mà:
	* allocate memory mới
	* Tạo string mới `left foot, right foot`
	* cho biến `s` trỏ sang memory mới, string cũ được gán vào `t` vẫn ko thay đổi
	* 
➡️String concatenation sẽ khiến chương trình ***cấp phát memory mới***, ***copy toàn bộ value cũ sang địa chỉ mới***.
💡Cách khắc phục: sử dụng `strings.Builder` 
*   `strings.Builder` có dạng `[]byte`, nên khi concate nó chỉ ***append vào buffer***, ***không recreate string, ít allocation, ít copy***.

## Một số thao tác (cheap) trên Strings
### Copy String
Do cấu trúc của string gần giống
```go
type string struct {
	ptr *byte
	len int
}
```
Nên khi copy string `t := s` thì chỉ cần copy `pointer` và `length` =>  không copy toàn bộ dữ liệu trong bộ nhớ

### Sub String
```go
s := "hello world"
sub := s[6:]
```
Khi sub string, Go ko allocate memory mới, mà chỉ di chuyển con trỏ của sub vào chung vùng nhớ với string ban đầu
```bash
Underlying byte array:
[h e l l o _ w o r l d]

s   -> ptr=0 len=11
sub -> ptr=6 len=5
```

## Benchmark strings
```go
package iteration  
  
import "strings"  
  
func Repeat(character string) string {  
    var repeatedString string  
  for i := 0; i < 5; i++ {  
       repeatedString += character  
    }  
    return repeatedString  
}  
  
func RepeatNoAllocation(character string) string {  
    var repeatedString strings.Builder  
  for i := 0; i < 5; i++ {  
       repeatedString.WriteString(character)  
    }  
    return repeatedString.String()  
}
```
```bash
go test .\... -bench ".*" -benchmem
```
Có kết quả
```bash
goos: windows
goarch: amd64
pkg: golearn/iteration
cpu: Intel(R) Core(TM) Ultra 7 155H
BenchmarkRepeat-22                      18593005                60.97 ns/op           16 B/op          4 allocs/op
BenchmarkRepeatNoAllocation-22          76488660                14.65 ns/op            8 B/op          1 allocs/op
```
* `B/op`:  là số lượng bytes đã allocated ở mỗi lần chạy (per iteration)
* `allocs/op`: là số lần ***memory allocations*** ở mỗi lần chạy (per iteration)

# Pointer
In Go, **when you call a function or a method the arguments are**  _**copied**_.
When calling `func (w Wallet) Deposit(amount int)` the `w` is a copy of whatever we called the method from.
```go

package pointer_error

import "fmt"

type Wallet struct {
	balance int
}

func (w Wallet) Deposit(amount int) int {
	fmt.Printf("address of balance in Deposit is %p \n", &w.balance)
	return w.balance + amount
}

func (w Wallet) Balance() int {
	return w.balance
}
```
```go
package pointer_error

import (
	"fmt"
	"testing"
)

func TestWallet(t *testing.T) {
	wallet := &Wallet{}

	wallet.Deposit(100)
	fmt.Printf("address of balance in test is %p \n", &wallet.balance)

	got := wallet.Balance()
	want := 100

	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA2MTUxODM5NCwyODAyNDM3ODVdfQ==
-->