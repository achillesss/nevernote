# Sudoku Solver In rust

### 数独：

#### 长相

常见的数独为九宫格：

```
	column	
	+---+---+---+---+---+---+---+---+---+
row	| 1 | 2 | 3 |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	| 4 | 5 | 6 |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	| 7 | 8 | 9 |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+
	|   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+

```

总的来说，数独是一个由81个小格子组成的 $9\times9$ 的大正方形。

#### 规则

九宫格规则很简单：

1. 每个小格子中填入 $[1,9]$ 中的任意一个数字
2. 每行的9个格子中没有重复的数字
3. 每列的9个格子中没有重复的数字
4. 每宫的9个格子中没有重复的数字

- 宫：

	整个数独有且仅有9个宫
	
	其中，对于常见的九宫格数独来说，一个``宫``为一个由9个小格子组成的$3\times3$的小正方形。第一个宫的样子由上图填满数字的9个格子组成，剩下的8个宫依次从左往右、从上往下排列。
	

### 准备

#### Position

首先，我们需要标识一下 81 个格子以及每行、每列、每宫的位置，这样才能区别开格子与格子、行与行、列与列、宫与宫。

``` rust
struct Position {
	x: u8,
	y: u8,
}
```

从左往右，格子坐标的 ``x`` 依次为 ``1 ~ 9``；从上往下，格子坐标的 ``y`` 依次为 ``1 ~ 9``

这样，格子的坐标就为：$[(1,1),(9,9)]$

对于行来说，我们只需要记住它们的 ``y`` 值，而对于列来说，记 ``x`` 值。所以，行的坐标为$[(x,1),(x,9)]$，列的坐标为$[(1,y),(9,y)]$。

对于一般的正方形宫来说，我们可以像规定格子的坐标一样来规定它们的坐标。于是，这种宫的坐标为$[(1,1),(3,3)]$。

于是，当我们知道一个格子的坐标为$(m,n)$时，我们就知道了它对应的行为$(x,n)$，对应的行为$(m,y)$，而对应的宫为$((m-1)/3+1,(n-1)/3+1)$。

接着，我们可能还需要：

- 判断两个``Position``是否相同
- 判断一个格子是否在一个宫内

```rust
pub struct Position {
    x: u8,
    y: u8,
}

impl Position {
    pub fn new(x: u8, y: u8) -> Position {
        Position { x: x, y: y }
    }

    pub fn x(&self) -> u8 {
        self.x
    }

    pub fn y(&self) -> u8 {
        self.y
    }

	// 判断两个位置是否一样
    pub fn equal(&self, position: &Self) -> bool {
        self.x == position.x && self.y == position.y
    }

	// 格子坐标转化为宫的坐标
    pub fn cell_to_square(&self) -> Position {
        Position::new((self.x - 1) / 3 + 1, (self.y - 1) / 3 + 1)
    }
	
	// 判断格子是否在宫内
    pub fn is_inner_cell(&self, position: &Position) -> bool {
        self.cell_to_square().equal(position)
    }
}
```

目前为止，看起来对于位置的处理方法已经差不多够用了。

#### Number

在求解数独时，我们往往会在一个没有填入确定数字的格子中预填一些数字，这些数字代表这个格子的所有可能值。

例如：

```
	+---+---+---+---+---+---+---+---+---+
	| 1 | 2 | 3 | 4 | 5 | 6 | 7 |   |   |
	+---+---+---+---+---+---+---+---+---+
	......
```

上面一行中，已经确定了前面7个格子的数字，剩下两个格子有可能出现的数字都为 ``8`` 或 ``9``。所以，一个格子中的值最多有可能达到9个。

一般情况下，我们其实可以直接定义一个这样的数字：

``` rust
struct SudokuNumber {
	number: Vec<u8>,
}
```

但是，我们也可以二般的这样定义一个数字：

```rust
struct SudokuNumber {
	nums: u16,
}
```

在这个 ``SudokuNumber`` 中，我们用一个 ``u16`` 来存放所有的数值。上面的一行，我们对应的格子中的值分别为：

``` rust
SudokuNumber { number: 1 }
SudokuNumber { number: 2 }
SudokuNumber { number: 4 }
SudokuNumber { number: 8 }
SudokuNumber { number: 16 }
SudokuNumber { number: 32 }
SudokuNumber { number: 64 }
SudokuNumber { number: 384 }
SudokuNumber { number: 384 }
```
转化成二进制就变成了：

``` rust
SudokuNumber { number: 0b1 }
SudokuNumber { number: 0b10 }
SudokuNumber { number: 0b100 }
SudokuNumber { number: 0b1000 }
SudokuNumber { number: 0b10000 }
SudokuNumber { number: 0b100000 }
SudokuNumber { number: 0b1000000 }
SudokuNumber { number: 0b110000000 }
SudokuNumber { number: 0b110000000 }
```

这样，一个格子中可能值是多少，就一目了然了。而且，对数值这样处理，会有很多很方便的地方：

- 增加一个数字：``self.number |= num``
- 减少一个数字：``self.number &= !num``


这样，数独数值的处理已经解决了。

``` rust
// 数字转化
pub fn decimal_to_binary(decimal_num: u32, check: bool) -> Option<u16> {
    let mut num = decimal_num;
    if num > 987654321 || num < 1 {
        if !check {
            return None;
        }
        num = 987654321;
    }


    let mut binary_num = 0u16;
    while num % 10 > 0 {
        binary_num |= 1 << ((num % 10) as u16 - 1);
        num /= 10;
    }

    Some(binary_num)
}

pub struct SudukuNumber {
    number: RefCell<u16>,
}

impl SudukuNumber {
    pub fn new(number: u32, check: bool) -> Self {
        let mut n = 0;
        if let Some(num) = decimal_to_binary(number, check) {
            n = num;
        }
        SudukuNumber { number: RefCell::new(n) }
    }

    // 增加数字
    pub fn add(&self, number: u32) -> &Self {
        if let Some(num) = decimal_to_binary(number, false) {
            *self.number.borrow_mut() |= num;
        }
        self
    }

    // 减少数字
    pub fn del(&mut self, number: u32) -> &mut Self {
        if let Some(num) = decimal_to_binary(number, false) {
            *self.number.borrow_mut() &= !num
        }
        self
    }

    // 转化成十进制后的数字组
    pub fn decimal(&self) -> Vec<u8> {
        let mut res = Vec::new();
        let mut dividend: u16 = 0b100000000;
        while dividend > 0 {
            if dividend & *self.number.borrow() != 0 {
                res.push(((dividend as f64).log2() + 1.0) as u8);
            }
            dividend >>= 1;
        }
        res
    }

    // 数字确定
    pub fn is_certain(&self) -> bool {
        self.decimal().len() == 1
    }
}
```

#### Cell

有了位置和数字，现在就可以定义一个格子了：

``` rust
pub struct Cell {
    numbers: SudukuNumber,
    position: Position,
}
```

解数独的根本就是解格子中的数字。所以，在这个``Cell``上，我们需要一系列的方法来对格子中的数字进行操作。

``` rust
impl Cell {
    pub fn new(position: Position) -> Cell {
        Cell {
            numbers: SudukuNumber::new(0, false),
            position: position,
        }
    }
	
	// 增加一个不确定的值
    pub fn add_number(&mut self, number: i16) -> &mut Self {
        self.numbers.add(number as i8);
        self
    }
	
	// 减少一个不确定的值
    pub fn del_number(&mut self, number: i16) -> &mut Self {
        self.numbers.del(number as i8);
        self
    }

	
    pub fn decimal_numbers(&self) -> Vec<u8> {
        self.numbers.decimal()
    }

    pub fn position(&self) -> &Position {
        &self.position
    }

    pub fn in_square(&self, square: Group) -> bool {
        match *square.form() {
            GroupForm::Square => self.position().is_inner_cell(square.position()),
            _ => false,
        }
    }

    pub fn satisfy_group(&self, group: &Group) -> bool {
        match group.form() {
            &GroupForm::Row => self.position().y() == group.position().y(),

            &GroupForm::Column => self.position().x() == group.position().x(),

            &GroupForm::Square => self.position().is_inner_cell(group.position()),
        }
    }

    pub fn is_in_group(&self, group: &Group) -> bool {
        self.satisfy_group(group) && group.not_empty(self.position())
    }
}
```

---

未完待续 ...