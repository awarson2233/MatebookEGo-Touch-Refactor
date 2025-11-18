| **阶段**      | **步骤**                          | **关键函数/操作**                                                                                                                              | **目的与注释**                                                      |
| ----------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **阶段一**     | 1.1                             | `if (DAT_18019538c > 1)`                                                                                                                 | **单例检查**：如果已初始化，则立即退出。                                         |
| **服务环境**    | 1.2                             | `CreateFileMappingA`                                                                                                                     | **IPC设置**：创建名为 `Global\HxIPCMemory` 和 `Global\HxIPCcmd` 的共享内存。 |
|             | 1.3                             | `CreateFileA` (日志文件)                                                                                                                     | **日志启动**：创建带时间戳的日志文件，句柄存入 `DAT_180195490`。                     |
|             | 1.4                             | `SetThreadPriority`                                                                                                                      | **提升优先级**：将当前线程设为 `THREAD_PRIORITY_TIME_CRITICAL` (15)。        |
| **阶段二**     | 2.1                             | `LoadLibraryW(L"SpiModule.x64.dll")`                                                                                                     | **加载HAL**：加载硬件抽象层 (HAL) DLL。                                   |
| **SPI HAL** | 2.2                             | `FUN_180046650` (GetProcAddress)                                                                                                         | **获取指针**：获取 DLL 中所有 `Ptr_...` 函数的地址。                           |
|             | 2.3 | `(*Ptr_SpbModuleInit)()`                                                                                                                 | **打开句柄**：初始化 SPI 模块，打开与内核驱动通信的设备句柄。                            |
|             | 2.4                             | `(*Ptr_SpiSetReset)`, `(*Ptr_SpiSetTimeout)`                                                                                             | **配置总线**：重置并配置 SPI 总线，使其进入已知状态。                                |
| **阶段三**     | 3.1                             | `FUN_180002920` (checkBusReady (himax_thp_drv!readRegister_ext_intf+0x6))                                                                | **Ping IC**：检查触摸 IC 是否响应 ("Bus dead" 检查)。                      |
| **硬件识别**    | 3.2                             | `addrIncUnitSetRaw_intf() (himax_thp_drv!safeModeSetRaw_intf+0x6e0)`, `burstModeSetRaw_intf() (himax_thp_drv!safeModeSetRaw_intf+0x790)` | **配置 IC 模式**：配置 IC 的 SPI 接口模式（如突发、地址递增）。                       |
|             | 3.3                             | `getProjectID_OTP()`                                                                                                                     | **读取 OTP 型号**：从 OTP 内存中读取硬件型号 (如 "W273AS2700")。                |
|             | 3.4                             | `strncmp`                                                                                                                                | **加载软件配置**：根据 OTP 型号，加载对应的软件配置指针。                              |
|             | 3.5                             | `FUN_18001eb00() 不使用设备`                                                                                                                  | **验证面板 ID**：交叉验证物理面板 ID 与 OTP 型号是否一致。                          |
| **阶段四**     | 4.1                             | `DAT_18019549c = *(... + 0x21500)`                                                                                                       | **加载“标准”版本**：从 `.exe` 内置固件 (`DAT_180195510`) 中读取“标准”固件版本号。     |
| **固件检查**    | 4.2                             | `FUN_180003da0('\x01') (himax_thp_drv!safeModeSetRaw_intf+0x6a0)`                                                                        | **准备 Flash 读取**：写入寄存器，命令 IC 准备好被读取 Flash。                      |
|             | 4.3                             | `flashCheckCRCRaw()`                                                                                                                     | **读取“当前” CRC**：从 Flash 读取固件并计算 CRC。结果存入 `bVar4`。               |
|             | 4.4                             | `hx_get_cfg_version_ahb()`                                                                                                               | **读取“当前”版本**：从 Flash 读取固件版本号。结果存入 `bVar2`。                     |
|             | 4.5                             | `if (...) goto LAB_180027a92`                                                                                                            | **决策**：如果 CRC 正确 **且** 版本一致，则**跳过**固件更新。                       |
|             | 4.6                             | `flashErase()`, `flashProgrammingInternal()`                                                                                             | **(如果需要) 更新固件**：擦除并刷写新固件，包含重试逻辑。                               |
| **阶段五**     | 5.1                             | `configParamByCfg()` (在 `LAB_180027a92`)                                                                                                 | **加载配置**：将 Flash 中的配置参数写入 IC 的工作寄存器。                           |
| **启动服务**    | 5.2                             | `Init_Buffers_And_Registers()`                                                                                                           | **初始化缓冲区**：初始化服务内部的内存缓冲区。                                      |
|             | 5.3                             | `(*Ptr_SpiIntOpen)(0)`, `(1)`                                                                                                            | **!! 启动服务 !!**：**告诉内核驱动开始监听触摸中断**。                             |
|             | 5.4                             | `DAT_18019538c = 4;`                                                                                                                     | **设置标志**：设置全局标志，标记初始化已成功。                                      |
| **阶段六**     | 6.1                             | `FUN_180046870(&local_3c8)` (在 `LAB_180028912`)                                                                                          | **内存清理**：释放用于存储旧日志列表的临时内存。                                     |
| **清理退出**    | 6.2                             | `__security_check_cookie`, `return`                                                                                                      | **函数退出**：安全检查并返回。                                              |

```batch
bp himax_thp_drv!thp_afe_open_project ".echo 'Enter!';"

::LoadSpiLibrary
bp kernel32!LoadLibraryW ".echo 'Load SPIModule!'; g;"

::addrIncUnitSetRaw_intf()
bp himax_thp_drv!safeModeSetRaw_intf+0x6e0 ".echo'Enter addrIncUnitSetRaw_intf!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x2bd2 ".echo 'Exit addrIncUnitSetRaw_intf!'; g;"

::burstModeSetRaw_intf()
bp himax_thp_drv!safeModeSetRaw_intf+0x790 ".echo'Enter burstModeSetRaw_intf!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x2e37 ".echo 'Exit burstModeSetRaw_intf!'; g;"

::getProjectID_OTP()
bp himax_thp_drv!getProjectID_OTP ".echo'Start getProjectID_OTP!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x32e7 ".echo 'Exit getProjectID_OTP!'; g;"

::FUN_180003da0
bp himax_thp_drv!safeModeSetRaw_intf+0x6a0 ".echo 'Enter FUN_180003da0!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x3a97 ".echo 'Exit FUN_180003da0!'; g;"

::flashCheckCRCRaw()
bp himax_thp_drv!flashCheckCRCRaw ".echo 'Enter flashCheckCRCRaw!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x3aba ".echo 'Exit flashCheckRCRaw!'; g;"

::hx_get_cfg_version_ahb
bp himax_thp_drv!hx_get_cfg_version_ahb ".echo 'Enter hx_get_cfg_version_ahb!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x3d0e ".echo 'Exit hx_get_cfg_version_ahb'; g;"

::configParamByCfg()
bp himax_thp_drv!configParamByCfg ".echo 'Enter configParamByCfg!'; g;"
bp himax_thp_drv!thp_afe_open_project+0x66ac ".echo 'Exit configParamByCfg!'; g;"

bp himax_thp_drv!readRegister_ext_intf+0x60 ".echo 'Enter checkBus!'; g;"

bp SpiModule.x64!SpiIntOpen ".printf \"\n[SpiIntOpenX64] DevID: %d\n\", @ecx; g;"
bp SpiModule.x64!SpiSetTimeout ".printf \"\n[SpiSetTimeoutX64] DevID: %d | Timeout(ms): %d\n\", @ecx, @edx; g;"
bp SpiModule.x64!SpiSetReset ".printf \"\n[SpiSetResetX64] DevID: %d | State: %d\n\", @ecx, @edx; g;"
bp SpiModule.x64!SpiSetBlock ".printf \"\n[SpiSetBlockX64] DevID: %d | State: %d\n\", @ecx, @edx; g;"
bp SpiModule_x64!SpiWrite ".printf \"\n[SpiWriteX64] DevID: %d | Ptr: 0x%I64x | Len: %I64d\n\", @ecx, @rdx, @r8; .echo '>>> DATA:'; db @rdx L@r8; g;"
bp /w "(@x1 >= 0x4001c00) && (@x1 <= 0x4001cFF)" Kernel32!DeviceIoControl ".printf \"[DeviceIoControl] Handle: 0x%x | IOCTL: 0x%x | InSize: %d | OutSize: %d\\n\", @x0, @x1, @x3, @x5; .if (@x3 > 0) { .echo '>>> INPUT DATA'; db @x2 L@x3; }; g;"

bp apdaemon!thpnotify+0x1496f ".echo'Exit!';"
```



```log
'Enter!'
himax_thp_drv!thp_afe_open_project:
00007ff8`fe7306b0 488bc4          mov     rax,rsp
0:013:AMD64> g
'Load SPIModule!'
ModLoad: 00007ff9`3ed10000 00007ff9`3ed66000   C:\Program Files\Huawei\HuaweiThpService\SpiModule.x64.dll
[DeviceIoControl] Handle: 0x58c | IOCTL: 0x4001c38 | InSize: 0 | OutSize: 4
[DeviceIoControl] Handle: 0x58c | IOCTL: 0x4001c38 | InSize: 0 | OutSize: 4
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c30 | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`464b4a88  01 00 00 00                                      ....
[DeviceIoControl] Handle: 0x728 | IOCTL: 0x4001c30 | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`464b4cc8  01 00 00 00                                      ....
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c2c | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`464b4a88  c8 00 00 00                                      ....
[DeviceIoControl] Handle: 0x728 | IOCTL: 0x4001c2c | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`464b4338  c8 00 00 00                                      ....
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c34 | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`464b4a88  00 00 00 00                                      ....
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c34 | InSize: 4 | OutSize: 0
'>>> INPUT DATA'
00000205`4648ca08  01 00 00 00                                      ....
'Enter checkBus!'
[DeviceIoControl] Handle: 0x728 | IOCTL: 0x4001c24 | InSize: 4 | OutSize: 4
'>>> INPUT DATA'
00000205`4648d6d0  f5 13 00 00                                      ....
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 4 | OutSize: 4
'>>> INPUT DATA'
00000205`4648d750  f3 13 00 00                                      ....
'Enter addrIncUnitSetRaw_intf!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6d0 | Len: 3 '>>> DATA:'
00000205`4648d6d0  f2 11 01                                         ...
'Exit addrIncUnitSetRaw_intf!'
'Enter burstModeSetRaw_intf!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d660 | Len: 3 '>>> DATA:'
00000205`4648d660  f2 13 31                                         ..1
'Exit burstModeSetRaw_intf!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d520 | Len: 3 '>>> DATA:'
00000205`4648d520  f2 0d 12                                         ...
'Start getProjectID_OTP!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7d0 | Len: 4 '>>> DATA:'
00000205`4648d7d0  f2 31 27 95                                      .1'.
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 5 | OutSize: 5
'>>> INPUT DATA'
00000205`4648d4d0  f3 31 00 00 00                                   .1...
[DeviceIoControl] Handle: 0x58c | IOCTL: 0x4001c38 | InSize: 0 | OutSize: 4
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b52b0 | Len: 10 '>>> DATA:'
00000205`464b52b0  f2 00 0c 80 00 90 53 ac-00 00                    ......S...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d520 | Len: 6 '>>> DATA:'
00000205`4648d520  f2 00 0c 80 00 90                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5c0 | Len: 3 '>>> DATA:'
00000205`4648d5c0  f2 0c 00                                         ...
ModLoad: 00000205`45bd0000 00000205`45bde000   image00000205`45bd0000
ModLoad: 00000205`45be0000 00000205`45bee000   image00000205`45be0000
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d650  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5130 | Len: 10 '>>> DATA:'
00000205`464b5130  f2 00 10 80 00 90 ca 35-00 00                    .......5..
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5250 | Len: 10 '>>> DATA:'
00000205`464b5250  f2 00 9c 00 00 90 dd 00-00 00                    ..........
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d650 | Len: 6 '>>> DATA:'
00000205`4648d650  f2 00 9c 00 00 90                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d670 | Len: 3 '>>> DATA:'
00000205`4648d670  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d700  f3 08 00 00 00 00 00                             .......
[DeviceIoControl] Handle: 0x58c | IOCTL: 0x4001c38 | InSize: 0 | OutSize: 4
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5290 | Len: 10 '>>> DATA:'
00000205`464b5290  f2 00 80 02 00 90 a5 00-00 00                    ..........
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d510 | Len: 6 '>>> DATA:'
00000205`4648d510  f2 00 80 02 00 90                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 3 '>>> DATA:'
00000205`4648d6e0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d4a0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5210 | Len: 10 '>>> DATA:'
00000205`464b5210  f2 00 00 b0 0e 30 eb 55-66 cc                    .....0.Uf.
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5270 | Len: 10 '>>> DATA:'
00000205`464b5270  f2 00 00 90 0b 30 b9 83-12 1a                    .....0....
[DeviceIoControl] Handle: 0x58c | IOCTL: 0x4001c38 | InSize: 0 | OutSize: 4
 [SpiWriteX64] DevID: 0 | Ptr: 0x205464b5250 | Len: 10 '>>> DATA:'
00000205`464b5250  f2 00 00 b0 0b 30 bb 02-0f 18                    .....0....
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d650 | Len: 8 '>>> DATA:'
00000205`4648d650  f2 00 00 b0 0b 30 bb 0a                          .....0..
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 3 '>>> DATA:'
00000205`4648d6e0  f2 0c 01                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d570 | Len: 3 '>>> DATA:'
00000205`4648d570  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d650 | Len: 6 '>>> DATA:'
00000205`4648d650  f2 00 01 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d780 | Len: 3 '>>> DATA:'
00000205`4648d780  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d650  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5b0 | Len: 3 '>>> DATA:'
00000205`4648d5b0  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d790 | Len: 3 '>>> DATA:'
00000205`4648d790  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 6 '>>> DATA:'
00000205`4648d7e0  f2 00 02 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 3 '>>> DATA:'
00000205`4648d7e0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d830  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 3 '>>> DATA:'
00000205`4648d6e0  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d620 | Len: 3 '>>> DATA:'
00000205`4648d620  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d780 | Len: 6 '>>> DATA:'
00000205`4648d780  f2 00 03 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d790 | Len: 3 '>>> DATA:'
00000205`4648d790  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d710  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d570 | Len: 3 '>>> DATA:'
00000205`4648d570  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d550 | Len: 3 '>>> DATA:'
00000205`4648d550  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 6 '>>> DATA:'
00000205`4648d500  f2 00 04 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d520 | Len: 3 '>>> DATA:'
00000205`4648d520  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d5e0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d790 | Len: 3 '>>> DATA:'
00000205`4648d790  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d550 | Len: 3 '>>> DATA:'
00000205`4648d550  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d700 | Len: 6 '>>> DATA:'
00000205`4648d700  f2 00 05 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d740 | Len: 3 '>>> DATA:'
00000205`4648d740  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d500  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 3 '>>> DATA:'
00000205`4648d500  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d740 | Len: 3 '>>> DATA:'
00000205`4648d740  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d770 | Len: 6 '>>> DATA:'
00000205`4648d770  f2 00 06 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d710 | Len: 3 '>>> DATA:'
00000205`4648d710  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d570  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d700 | Len: 3 '>>> DATA:'
00000205`4648d700  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5e0 | Len: 3 '>>> DATA:'
00000205`4648d5e0  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d740 | Len: 6 '>>> DATA:'
00000205`4648d740  f2 00 07 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 3 '>>> DATA:'
00000205`4648d7e0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d5b0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 3 '>>> DATA:'
00000205`4648d7e0  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5b0 | Len: 3 '>>> DATA:'
00000205`4648d5b0  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d620 | Len: 6 '>>> DATA:'
00000205`4648d620  f2 00 08 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d760 | Len: 3 '>>> DATA:'
00000205`4648d760  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d620  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 3 '>>> DATA:'
00000205`4648d6e0  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d550 | Len: 3 '>>> DATA:'
00000205`4648d550  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 6 '>>> DATA:'
00000205`4648d500  f2 00 09 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d520 | Len: 3 '>>> DATA:'
00000205`4648d520  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d5e0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d790 | Len: 3 '>>> DATA:'
00000205`4648d790  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5e0 | Len: 3 '>>> DATA:'
00000205`4648d5e0  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 6 '>>> DATA:'
00000205`4648d500  f2 00 0a 1c 0b 30                                .....0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 3 '>>> DATA:'
00000205`4648d500  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d790  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d620 | Len: 3 '>>> DATA:'
00000205`4648d620  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d3e0 | Len: 18 '>>> DATA:'
00000205`4648d3e0  f2 00 00 b0 0b 30 00 02-00 00 00 00 00 00 00 70  .....0.........p
00000205`4648d3f0  00 0b                                            ..
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5b0 | Len: 3 '>>> DATA:'
00000205`4648d5b0  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d700 | Len: 6 '>>> DATA:'
00000205`4648d700  f2 00 08 40 0b 30                                ...@.0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d780 | Len: 3 '>>> DATA:'
00000205`4648d780  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d770  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d4d0 | Len: 3 '>>> DATA:'
00000205`4648d4d0  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d700 | Len: 3 '>>> DATA:'
00000205`4648d700  f2 0d 12                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d830 | Len: 6 '>>> DATA:'
00000205`4648d830  f2 00 01 20 0e 30                                ... .0
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d500 | Len: 3 '>>> DATA:'
00000205`4648d500  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d6e0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d530 | Len: 3 '>>> DATA:'
00000205`4648d530  f2 0d 13                                         ...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d280 | Len: 10 '>>> DATA:'
00000205`4648d280  f2 00 80 02 00 90 00 00-00 00                    ..........
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5b0 | Len: 6 '>>> DATA:'
00000205`4648d5b0  f2 00 80 02 00 90                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d510 | Len: 3 '>>> DATA:'
00000205`4648d510  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d550  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d280 | Len: 10 '>>> DATA:'
00000205`4648d280  f2 00 9c 00 00 90 00 00-00 00                    ..........
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 6 '>>> DATA:'
00000205`4648d7e0  f2 00 9c 00 00 90                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5b0 | Len: 3 '>>> DATA:'
00000205`4648d5b0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d790  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 4 '>>> DATA:'
00000205`4648d6e0  f2 31 00 00                                      .1..
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 5 | OutSize: 5
'>>> INPUT DATA'
00000205`4648d5b0  f3 31 00 00 00                                   .1...
'Exit getProjectID_OTP!'
'Enter FUN_180003da0!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d570 | Len: 4 '>>> DATA:'
00000205`4648d570  f2 31 27 95                                      .1'.
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 5 | OutSize: 5
'>>> INPUT DATA'
00000205`4648d500  f3 31 00 00 00                                   .1...
'Exit FUN_180003da0!'
'Enter flashCheckCRCRaw!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d780 | Len: 4 '>>> DATA:'
00000205`4648d780  f2 31 27 95                                      .1'.
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 5 | OutSize: 5
'>>> INPUT DATA'
00000205`4648d550  f3 31 00 00 00                                   .1...
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d0a0 | Len: 10 '>>> DATA:'
00000205`4648d0a0  f2 00 40 00 00 80 02 00-00 00                    ..@.......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d2a0 | Len: 10 '>>> DATA:'
00000205`4648d2a0  f2 00 20 00 05 80 00 00-00 00                    .. .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d260 | Len: 10 '>>> DATA:'
00000205`4648d260  f2 00 28 00 05 80 00 ff-99 00                    ..(.......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d5f0 | Len: 6 '>>> DATA:'
00000205`4648d5f0  f2 00 00 00 05 80                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d570 | Len: 3 '>>> DATA:'
00000205`4648d570  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d700  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d740 | Len: 6 '>>> DATA:'
00000205`4648d740  f2 00 00 00 05 80                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7e0 | Len: 3 '>>> DATA:'
00000205`4648d7e0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d5b0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d6e0 | Len: 6 '>>> DATA:'
00000205`4648d6e0  f2 00 00 00 05 80                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d570 | Len: 3 '>>> DATA:'
00000205`4648d570  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d6e0  f3 08 00 00 00 00 00                             .......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d590 | Len: 6 '>>> DATA:'
00000205`4648d590  f2 00 18 00 05 80                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d7c0 | Len: 3 '>>> DATA:'
00000205`4648d7c0  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d570  f3 08 00 00 00 00 00                             .......
'Exit flashCheckRCRaw!'
'Enter hx_get_cfg_version_ahb!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d780 | Len: 6 '>>> DATA:'
00000205`4648d780  f2 00 84 70 00 10                                ...p..
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d670 | Len: 3 '>>> DATA:'
00000205`4648d670  f2 0c 00                                         ...
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c24 | InSize: 7 | OutSize: 7
'>>> INPUT DATA'
00000205`4648d700  f3 08 00 00 00 00 00                             .......
'Exit hx_get_cfg_version_ahb'
'Enter configParamByCfg!'
'Exit configParamByCfg!'
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054647ffa0 | Len: 86 '>>> DATA:'
00000205`4647ffa0  f2 00 50 75 00 10 00 00-00 00 00 00 00 00 00 00  ..Pu............
00000205`4647ffb0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000205`4647ffc0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000205`4647ffd0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000205`4647ffe0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000205`4647fff0  00 00 00 00 00 00                                ......
 [SpiWriteX64] DevID: 0 | Ptr: 0x2054648d280 | Len: 10 '>>> DATA:'
00000205`4648d280  f2 00 3c 75 00 10 00 00-00 00                    ..<u......
[DeviceIoControl] Handle: 0x3f8 | IOCTL: 0x4001c00 | InSize: 0 | OutSize: 0
[DeviceIoControl] Handle: 0x728 | IOCTL: 0x4001c00 | InSize: 0 | OutSize: 0
'Exit!'
ApDaemon!ThpNotify+0x1496f:
00007ff9`3e278e7f 488b0f          mov     rcx,qword ptr [rdi] ds:00000205`4647a538=000002054647a518

```
