# Writing a Chip-8 Emulator in Rust

The [Chip-8](https://en.wikipedia.org/wiki/CHIP-8) system is whats known as a fantasy console, which roughly means that it wasn’t made to be run directly on real hardware. Instead, even at its first implementation around 50 years ago it was emulated. In my experience it is commonly used as the “Hello world” of console emulation. Because it was being emulated on such old hardware, even python is more than fast enough, but since Rust is so strict with things like data types it will help prevent annoying bugs. I’m choosing Rust over C++ or similar simply because I am better at it.

One way to get started is to write a basic but empty framework for the emulator and write a simple test, then try to get the test to not immediatly fail.

```rust
// lib.rs

#[cfg(test)]
mod tests;

pub struct Chip8{

}

impl Chip8{
  pub fn new()->Self{
    Self{}
  }
  pub fn step(&mut self){
    todo!()
  }
}
```

This code just establishes a struct for the system, allows it to be made with ``` chip8::new() ```, which is standard, and creates a function ``` step() ``` that will eventually execute a single instruction, but currently just crashes the program.

Next, we will need a test. This test runs a program that should just display the IBM logo for 700 steps. It is a fairly standard program for starting off because it requires only a few instructions to be implemented. 700 steps is probably overkill, but its about 1 second of execution time, though there isn't a precise standard for that. The rom that I downloaded is located at https://github.com/loktar00/chip8 and called "IBM Logo.ch8" but can be found elsewhere. If you use a different copy, it might have a slightly different result and/or need different instructions. The code on this page assumes that the rom is placed in a folder called test_roms inside the src folder.

```rust
// tests.rs

use super::*;

#[test]
fn test_ibm_logo_rom(){
  let mut chip = Chip8::new();
  for _ in 0..700{
    chip.step();
  }
}
```

If you run this code now, using ``` cargo t ``` it will fail the test, saying "not yet implemented" because the step function just has todo!() in it. Next, we will add the main three steps involved in emulation: fetch, decode, execute.

Fetch means to get the opcode that will be executed, meaning the byte that controls what the computer will do next. This is tracked with a program counter, which we will increment in this stage, it could be done later, but this step is already simple enough.

Decode means to convert the opcode to a more easily understandable instruction. The Chip-8 system is simple enough, eg there is only one opcode that adds two number together, so this stage can be merged with the next.

Execute takes the instruction and does whatever it says. Since we are merging this with decode, we will use the Rust's match statement for the opcode and run it right there.

We will also need somewhere for the rom being run to be stored, and we will change the ```new()``` function to take the rom and the step function to combine fetch and execute. 

Finally, we will also need a function for reading 16 bit values. This is because the opcodes are 16 bit.

```rust
// lib.rs

...

pub struct Chip8 {
    ram: [u8; 0xFFF],
    program_counter: u16,
}

impl Chip8 {
    pub fn new(rom: &[u8]) -> Self {
        let mut ram = [0; 0xFFF];
        ram[0x200..0x200 + rom.len()].copy_from_slice(rom);

        Self {
            ram,
            program_counter: 0x200,
        }
    }
    pub fn step(&mut self) {
        let opcode = self.fetch();
        self.execute(opcode);
    }

    fn fetch(&mut self) -> u16 {
        let opcode = self.read_16(self.program_counter);
        self.program_counter += 2;
        opcode
    }
    fn execute(&mut self, opcode: u16){
        match opcode{
            _ => todo!("Opcode 0x{opcode:04X)"),
        }
    }

    fn read_16(&self, addr: u16)->u16{
        ((self.ram[addr as usize] as u16) << 8) | (self.ram[addr as usize+1] as u16)
    }
}
```

We also need to update the test code to load the ibm logo rom.

```rust
...
fn test_ibm_logo_rom(){
  let mut chip = Chip8::new(include_bytes!("test_roms/IBM Logo.ch8"));
  ...
}
```

If we run the test now it... still fails. But! the todo call will now print the opcode that is trying to be executed. Now, we can add new lines to respond to the failure in the test.

The first opcode is 0x00E0, which clears the screen. It is important for the emulator the be able to see its own screen, because later instructions will change the emulators state based on the screen's state. The Chip-8 originally had a 64x32 pixel display with on or off states, so we will make a 2d array of booleans to store the screen state.

```rust
pub struct Chip8 {
    ...
    screen: [[bool; 32]; 64],
    ...
}

impl Chip8{
    ...
    fn execute(&mut self, opcode: u16){
        match opcode{
            0x00E0 => self.screen = [[false; 32]; 64],
            _ => todo!("Opcode 0x{opcode:04X}"),
        }
    }
    ...
}
```

This is mostly how ading new instructions will go, with the exception that some will need new functions and we haven't added the registers, the stack, input, or timers.

The next instruction is 0xA22A. Instructions that start with A means to set the I register, aka the address register, to the lower 12 bits, 0x22A. The I register is technically 12 bit, but Rust doesn't allow that, so we'll use u16. We can use the range syntax to match all A values and a mask to get the lower 12 bits of the opcode.

```rust
pub struct Chip8{
    ...
    address_register: u16
}

impl Chip8{
    pub fn new(rom: &[u8]) -> Self{
        ...
        Self{
            ...
            address_register: 0
        }
    }
    fn execute(&mut self, opcode: u16){
        ...
            0xA000..=0xAFFF => self.address_register = opcode&0x0FFF,
        ...
    }
}
```

Next is the opcode 0x600C. The 0x6000 block sets register A to value NN where the opcode defines A and NN like 0x6ANN. So the instruction sets register 0 to 0x0C. There are 16 other registers, all 8 bit, v0 to vF.

To make this easier, we will write two helper functions, ``` get_x(opcode) ``` and ``` get_nn(opcode) ``` that just take the correct part of the opcode. The registers also need to be added, so we can just use an array.

```rust
pub struct Chip8 {
    ...
    registers: [u8; 16],
    ...
}

impl Chip8 {
    ...
    const fn get_x(opcode: u16) -> u8{
        ((opcode & 0x0F00) >> 8) as u8
    }
    const fn get_nn(opcode: u16) -> u8{
        (opcode & 0x00FF) as u8
    }
}
```

And the actual instruction can be added to the opcode like before.

```rust
fn execute(&mut self, opcode: u16){
    match opcode{
        ...
        0x6000..=0x6FFF => self.registers[Self::get_x(opcode) as usize] = Self::get_nn(opcode),
        ...
    }
}
```

Next is probably the most complex instruction: 0xD01F, part of the 0xD000 block which controls the display. We will save that for the end since its so complicated. The instruction can be skipped if we match the opcode but don't do anything.

```rust
0xD000..=0xDFFF => (),
```
So now for 0x7009. The 0x7000 block is like the 0x6000 block, but instead it adds rather than assigns. To prevent an overflow error, we will use wrapping_add instead of a normal add.

```rust
0x7000..=0x7FFF => self.registers[Self::get_x(opcode) as usize] = self.registers[Self::get_x(opcode) as usize].wrapping_add(Self::get_nn(opcode)),
```

Finally, 0x1228. The 0x1000 block gets the lower 12 bits and jumps to that address. We can make another helper function to help keep the code clean, ```get_nnn(opcode)```.

```rust
impl Chip8 {
    ...
    const fn get_nnn(opcode: u16) -> u16{
        opcode & 0x0FFF
    }
}
```

And now adding the 0x1000 block is simple.

```rust
0x1000..=0x1FFF => self.program_counter = Self::get_nnn(opcode),
```

Now, if you run the test again, it actually passes. Of course, we need to add the draw code before we can actually see the logo. The draw function will draw sprites from memory and hopefully the exact method will become clear. The first thing we want is some more helper functions.

```rust
const fn get_y(opcode: u16) -> u8{
    ((opcode & 0x00F0) >> 4) as u8
}

const fn get_n(opcode: u16) -> u8{
    (opcode & 0x000F) as u8
}
```
Now we can start the actual function. Since it's a bit long, I will break it up and explain as I go.

We need to get the location for the sprite and its height. The width is always 8 pixels each call, though sprites can be wider or narrower. We also need to wrap around the screen so the drawing always starts on the screen.
```rust
let x = self.registers[Self::get_x(opcode) as usize] as usize % 64;
let y = self.registers[Self::get_y(opcode) as usize] as usize % 32;
let height = Self::get_n(opcode) as usize;
```

Next we set the VF register to 0. After a call to draw, the VF register will be 1 if any pixel was turned off and 0 otherwise. The sprite data just decides what pixels to toggle, so if the pixel is already on, it turns off.

```rust
self.registers[0xF] = 0;
```

Now, we will loop through the height of the pixel, then check if we are drawing off the screen and if we are we just stop.

```rust
for n in 0..height{
    if y + n >= 32{
        return;
    }
```

Now we get the byte that represents this layer of the sprite. The fist layer is located at the address register and we just add n to get the following layers.

```rust
    let byte = self.ram[self.address_register as usize + n];
```

We also need to loop through the bits in the byte like we did with the height and stop drawing the layer if it goes off screen.

```rust
    for bit_index in 0..7{
        if x + bit_index >= 64{
            continue;
        }
```

Now we get the bit in the byte. We start at the most signifigant and go down.

```rust
        let bit = byte & 1<<(7-bit_index) != 0;
```

Finally, if the bit is on, we check the pixel and if its on we turn it off and set VF to 1, otherwise we just turn it on.

```rust
if bit{
        if self.screen[x+bit_index][y+n]{
            self.screen[x+bit_index][y+n] = false;
            self.registers[0xF] = 1;
        } else {
            self.screen[x+bit_index][y+n] = true;
        }
```

And heres the fuction in its entirety.
 
 ```rust
fn draw(&mut self, opcode: u16){
    let x = self.registers[Self::get_x(opcode) as usize] as usize % 64;
    let y = self.registers[Self::get_y(opcode) as usize] as usize % 32;
    let height = Self::get_n(opcode) as usize;
    self.registers[0xF] = 0;
    for n in 0..height{
        if y + n >= 32{
            return;
        }
        let byte = self.ram[self.address_register as usize + n];
        for bit_index in 0..8{
            if x + bit_index >= 64{
                continue;
            }

            let bit = byte & 1<<(7-bit_index) != 0;
            if bit{
                if self.screen[x+bit_index][y+n]{
                    self.screen[x+bit_index][y+n] = false;
                    self.registers[0xF] = 1;
                } else {
                    self.screen[x+bit_index][y+n] = true;
                }
            }
        }
    }
}
```
Don't forget we need to call the function in the 0x0xD000 opcode block.

We can see what the result by temperarily modifying the test function to show the display, then intentionally panic.

```rust
// test.rs
use super::*;

#[test]
fn test_ibm_logo_rom(){
    let mut chip = Chip8::new(include_bytes!("test_roms/IBM Logo.ch8"));
    for _ in 0..700{
        chip.step();
    }
    for y in 0..32{
        for x in 0..64{
            if chip.screen[x][y]{
                print!("█")
            } else {
                print!{" "}
            }
        }
        println!()
    }
    panic!();
}
```

It should look like this, except I added the frame to show the spacing
```
------------------------------------------------------------------
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|            ████████ █████████   █████         █████            |
|                                                                |
|            ████████ ███████████ ██████       ██████            |
|                                                                |
|              ████     ███   ███   █████     █████              |
|                                                                |
|              ████     ███████     ███████ ███████              |
|                                                                |
|              ████     ███████     ███ ███████ ███              |
|                                                                |
|              ████     ███   ███   ███  █████  ███              |
|                                                                |
|            ████████ ███████████ █████   ███   █████            |
|                                                                |
|            ████████ █████████   █████    █    █████            |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
|                                                                |
------------------------------------------------------------------
```

We will actually use this to finish off the test. Instead of printing it will compare with a referance to check each pixel.

```rust
    const TARGET: [&str; 32] = [
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "            ████████ █████████   █████         █████            ",
        "                                                                ",
        "            ████████ ███████████ ██████       ██████            ",
        "                                                                ",
        "              ████     ███   ███   █████     █████              ",
        "                                                                ",
        "              ████     ███████     ███████ ███████              ",
        "                                                                ",
        "              ████     ███████     ███ ███████ ███              ",
        "                                                                ",
        "              ████     ███   ███   ███  █████  ███              ",
        "                                                                ",
        "            ████████ ███████████ █████   ███   █████            ",
        "                                                                ",
        "            ████████ █████████   █████    █    █████            ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
        "                                                                ",
    ];

    ...

    for y in 0..32{
        for x in 0..64{
            if chip.screen[x][y]{
                assert_eq!(TARGET[y].chars().nth(x).unwrap(), '█');
            } else {
                assert_eq!(TARGET[y].chars().nth(x).unwrap(), ' ');
            }
        }
    }
```

And thats all! At least for now. There are many more opcodes that need to be implemented, but this is a good start, and a good kickoff point for someone who wants to try the rest themself!

If your having any problems, my version is shown below.

```rust
#[cfg(test)]
mod tests;

pub struct Chip8 {
    ram: [u8; 0xFFF],
    screen: [[bool; 32]; 64],
    registers: [u8; 16],
    program_counter: u16,
    address_register: u16,
}

impl Chip8 {
    pub fn new(rom: &[u8]) -> Self {
        let mut ram = [0; 0xFFF];
        ram[0x200..0x200 + rom.len()].copy_from_slice(rom);

        Self {
            ram,
            screen: [[false; 32]; 64],
            registers: [0; 16],
            program_counter: 0x200,
            address_register: 0,
        }
    }
    pub fn step(&mut self) {
        let opcode = self.fetch();
        self.execute(opcode);
    }

    fn fetch(&mut self) -> u16 {
        let opcode = self.read_16(self.program_counter);
        self.program_counter += 2;
        opcode
    }
    fn execute(&mut self, opcode: u16){
        match opcode{
            0x00E0 => self.screen = [[false; 32]; 64],
            0x1000..=0x1FFF => self.program_counter = Self::get_nnn(opcode),
            0x6000..=0x6FFF => self.registers[Self::get_x(opcode) as usize] = Self::get_nn(opcode),
            0x7000..=0x7FFF => self.registers[Self::get_x(opcode) as usize] = self.registers[Self::get_x(opcode) as usize].wrapping_add(Self::get_nn(opcode)),
            0xA000..=0xAFFF => self.address_register = opcode&0x0FFF,
            0xD000..=0xDFFF => self.draw(opcode),
            _ => todo!("Opcode 0x{opcode:04X}"),
        }
    }

    const fn get_x(opcode: u16) -> u8{
        ((opcode & 0x0F00) >> 8) as u8
    }
    const fn get_y(opcode: u16) -> u8{
        ((opcode & 0x00F0) >> 4) as u8
    }
    const fn get_n(opcode: u16) -> u8{
        (opcode & 0x000F) as u8
    }
    const fn get_nn(opcode: u16) -> u8{
        (opcode & 0x00FF) as u8
    }
    const fn get_nnn(opcode: u16) -> u16{
        opcode & 0x0FFF
    }
    
    fn read_16(&self, addr: u16)->u16{
        ((self.ram[addr as usize] as u16) << 8) | (self.ram[addr as usize+1] as u16)
    }

    fn draw(&mut self, opcode: u16){
        let x = self.registers[Self::get_x(opcode) as usize] as usize % 64;
        let y = self.registers[Self::get_y(opcode) as usize] as usize % 32;
        let height = Self::get_n(opcode) as usize;
        self.registers[0xF] = 0;
        for n in 0..height{
            if y + n >= 32{
                return;
            }
            let byte = self.ram[self.address_register as usize + n];
            for bit_index in 0..8{
                if x + bit_index >= 64{
                    continue;
                }

                let bit = byte & 1<<(7-bit_index) != 0;
                if bit{
                    if self.screen[x+bit_index][y+n]{
                        self.screen[x+bit_index][y+n] = false;
                        self.registers[0xF] = 1;
                    } else {
                        self.screen[x+bit_index][y+n] = true;
                    }
                }
            }
        }
    }
}
```
