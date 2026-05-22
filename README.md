# CPU 8-bit Custom — Verilog

> 🚧 **Work in Progress** — Dự án đang trong quá trình phát triển. Các module cơ bản đã hoàn thành và chạy được; các tính năng nâng cao (UART, Stack, FPGA synthesis) đang tiếp tục được thêm vào.

Dự án thiết kế một CPU 8-bit tùy chỉnh từ đầu bằng Verilog, phục vụ mục đích học tập kiến trúc máy tính. CPU thực thi vòng lặp **Fetch → Decode → Execute** với tập lệnh (ISA) tự thiết kế, chạy được chương trình Assembly thực tế và có thể simulate bằng iverilog + GTKWave.

---

## Demo

Chương trình mẫu: tính tổng **1 + 2 + 3 + 4 + 5 = 15**

```
Cycle |  State  | PC |   INSTR  | R0  R1  R2  R3 | Z C
------|---------|----+----------+----------------+----
    5 | EXECUTE |  1 | 00000000 |  0   x   x   x | 0 0   ← LDI R0, 0
    8 | EXECUTE |  2 | 00001001 |  0   1   x   x | 0 0   ← LDI R1, 1
   17 | EXECUTE |  5 | 01100010 |  1   1   5   1 | 0 0   ← ADD R0, R1 (lần 1)
   ...
  ??? | EXECUTE | 12 | 01000000 | 15   6   0   1 | 1 0   ← ST mem[0]=15, DONE
```

---

## Kiến trúc

### Instruction Format — 8-bit fixed-length

```
 7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
|  opcode (3) |  Ra (2) | Rb/imm3 |
+---+---+---+---+---+---+---+---+
```

| Field | Bits | Mô tả |
|-------|------|-------|
| opcode | [7:5] | Loại lệnh (8 lệnh) |
| Ra | [4:3] | Register đích / nguồn A |
| Rb | [2:1] | Register nguồn B |
| imm3 | [2:0] | Immediate 3-bit cho LDI (0–7) |
| imm5 | [4:0] | Địa chỉ 5-bit cho JMP/JZ/ST/LD |

### Tập lệnh (ISA)

| Opcode | Mnemonic | Ý nghĩa |
|--------|----------|---------|
| `000` | `LDI Ra, #imm` | Ra ← imm (0–7) |
| `001` | `LD Ra, [addr]` | Ra ← mem[addr] |
| `010` | `ST [addr], Ra` | mem[addr] ← Ra |
| `011` | `ADD Ra, Rb` | Ra ← Ra + Rb |
| `100` | `SUB Ra, Rb` | Ra ← Ra − Rb |
| `101` | `AND Ra, Rb` | Ra ← Ra & Rb |
| `110` | `JZ addr` | PC ← addr nếu Z=1 |
| `111` | `JMP addr` | PC ← addr |

**Registers:** R0–R3 (4 thanh ghi × 8-bit)  
**Flags:** Zero (Z), Carry (C)  
**Memory:** Harvard architecture — Instruction ROM 256B + Data RAM 256B

### Datapath

```
        ┌──────┐   instr   ┌─────────┐  opcode  ┌─────────┐
 PC ───▶│ IMEM │──────────▶│ Decoder │─────────▶│ Control │
        └──────┘           └─────────┘  signals  └─────────┘
                                │ Ra, Rb               │
                                ▼                       │ reg_we
                           ┌─────────┐                  │ mem_we
                           │ RegFile │◀─────────wb_data─┘
                           └─────────┘
                             │     │ Ra, Rb
                             ▼     ▼
                           ┌─────────┐  result
                           │   ALU   │──────────▶ WB MUX ──▶ RegFile
                           └─────────┘
                             │  Z, C
                             ▼
                           ┌─────────┐
                           │  Flags  │──▶ Control (cho JZ/JC)
                           └─────────┘
```

---

## Cấu trúc thư mục

```
cpu8bit/
├── rtl/                   # Source Verilog
│   ├── alu.v              # ALU 8-bit (ADD/SUB/AND/OR/XOR/NOT/INC/DEC)
│   ├── regfile.v          # Register File 4×8-bit
│   ├── pc.v               # Program Counter
│   ├── flags.v            # Zero & Carry Flag Register
│   ├── imem.v             # Instruction Memory (ROM)
│   ├── dmem.v             # Data Memory (RAM)
│   ├── decoder.v          # Instruction Decoder
│   ├── control.v          # Control Unit (FSM 3-state)
│   └── cpu.v              # Top-level CPU
├── tb/                    # Testbenches
│   ├── tb_alu.v           # Test ALU đơn lẻ
│   ├── tb_regfile.v       # Test Register File
│   ├── tb_pc_flags.v      # Test PC + Flags
│   ├── tb_phase2.v        # Test Decoder + Control Unit
│   └── tb_cpu.v           # Testbench top-level (full simulation)
├── prog/                  # Chương trình
│   ├── assembler.py       # Assembler Python: .asm → .hex
│   └── prog.hex           # Chương trình mẫu (tổng 1..5)
├── sim/                   # Output simulation (tạo trước khi chạy)
└── README.md
```

---

## Chạy simulation

### Yêu cầu

- [iverilog](https://bleyer.org/icarus/) (Icarus Verilog)
- [GTKWave](https://gtkwave.sourceforge.net/) (xem waveform, tuỳ chọn)
- Python 3 (để dùng assembler)

Cài trên Ubuntu/Debian:
```bash
sudo apt install iverilog gtkwave
```

Cài trên macOS:
```bash
brew install icarus-verilog gtkwave
```

### Chạy full CPU simulation

```bash
# 1. Tạo thư mục output
mkdir -p sim

# 2. Compile
iverilog -g2012 -o sim/tb_cpu tb/tb_cpu.v rtl/*.v

# 3. Chạy simulation
vvp sim/tb_cpu

# 4. Xem waveform (tuỳ chọn)
gtkwave sim/cpu.vcd
```

**Kết quả mong đợi:**
```
KẾT QUẢ KIỂM TRA:
  PASS  R0 = 15 (mong đợi 15 = 1+2+3+4+5)
  PASS  R1 = 6 (counter dừng tại 6)
  PASS  R2 = 0 (đếm ngược về 0)
  PASS  mem[0] = 15 (kết quả đã lưu vào memory)
  PASS  Zero flag = 1
```

### Chạy test từng module

```bash
# ALU
iverilog -g2012 -o sim/tb_alu tb/tb_alu.v rtl/alu.v && vvp sim/tb_alu

# Register File
iverilog -g2012 -o sim/tb_regfile tb/tb_regfile.v rtl/regfile.v && vvp sim/tb_regfile

# PC + Flags
iverilog -g2012 -o sim/tb_pc_flags tb/tb_pc_flags.v rtl/pc.v rtl/flags.v && vvp sim/tb_pc_flags

# Decoder + Control Unit
iverilog -g2012 -o sim/tb_phase2 tb/tb_phase2.v rtl/decoder.v rtl/control.v && vvp sim/tb_phase2
```

### Viết chương trình mới

```bash
# Sửa chương trình trong assembler.py (biến PROGRAM)
# hoặc tạo file .asm riêng:
python3 prog/assembler.py myprogram.asm > prog/prog.hex

# Sau đó compile lại và chạy như bình thường
```

---

## Giai đoạn phát triển

| Giai đoạn | Nội dung | Modules | Trạng thái |
|-----------|----------|---------|------------|
| 1 | Các khối cơ bản | `alu.v`, `regfile.v`, `pc.v`, `flags.v` | ✅ Hoàn thành |
| 2 | Memory & Decoder | `imem.v`, `dmem.v`, `decoder.v`, `control.v` | ✅ Hoàn thành |
| 3 | Ghép nối & chạy | `cpu.v`, `tb_cpu.v`, `prog.hex` | ✅ Hoàn thành |
| 4 | Mở rộng & nâng cao | UART, Stack, FPGA | 🚧 Đang làm |

---

## Lệnh ALU đầy đủ

| ALU op | Mã | Phép toán |
|--------|----|-----------|
| ADD | `000` | A + B |
| SUB | `001` | A − B |
| AND | `010` | A & B |
| OR  | `011` | A \| B |
| XOR | `100` | A ^ B |
| NOT | `101` | ~A |
| INC | `110` | A + 1 |
| DEC | `111` | A − 1 |

---

## Hướng phát triển tiếp theo

- [x] ALU 8-bit với đầy đủ phép toán
- [x] Register File, Program Counter, Flag Register
- [x] Instruction Memory, Data Memory, Decoder, Control FSM
- [x] Top-level CPU + testbench + chương trình mẫu
- [ ] Thêm lệnh `OR`, `XOR`, `NOT` vào ISA
- [ ] Stack pointer + lệnh `PUSH`/`POP`/`CALL`/`RET`
- [ ] UART TX output — in kết quả ra terminal
- [ ] Assembler hỗ trợ label và comment đầy đủ hơn
- [ ] Synthesis lên FPGA (Xilinx/Altera), hiển thị kết quả qua LED/7-seg

---

## Tài liệu tham khảo

- Patterson & Hennessy — *Computer Organization and Design*
- [Icarus Verilog documentation](https://steveicarus.github.io/iverilog/)
- [GTKWave User Guide](https://gtkwave.sourceforge.net/gtkwave.pdf)

---

## License

MIT License — tự do sử dụng cho mục đích học tập và nghiên cứu.
