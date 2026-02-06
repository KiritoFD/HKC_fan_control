
# HKC 笔记本风扇控制

作者：ArcherFD

> 通过 ACPI WMI 接口对 HKC 笔记本风扇档位进行控制的逆向笔记与可用命令记录。

---

## 快速使用（PowerShell）

**自动 / BIOS 控制**
```powershell
Get-CimInstance -Namespace root\wmi -ClassName EmdAcpi_FanControl |
Invoke-CimMethod -MethodName FanControl -Arguments @{ Data = 0 }
````

**静音模式（最低固定转速）**

```powershell
Get-CimInstance -Namespace root\wmi -ClassName EmdAcpi_FanControl |
Invoke-CimMethod -MethodName FanControl -Arguments @{ Data = 1 }
```

**性能模式（最高可用档位）**

```powershell
Get-CimInstance -Namespace root\wmi -ClassName EmdAcpi_FanControl |
Invoke-CimMethod -MethodName FanControl -Arguments @{ Data = 2 }
```

> 说明：这是 BIOS 层的档位切换，并非连续 PWM 调速。

---

## 背景

在 HKC 笔记本上，常见的第三方风扇控制工具（如 NBFC、FanControl）默认无法识别风扇控制接口。
通过检查 ACPI 表和 WMI 设备，发现厂商提供了隐藏的 WMI 控制通道，可用于切换风扇工作模式。

---

## 温度接口的局限性

系统中可见标准 ACPI Thermal Zone：

```powershell
Get-CimInstance -Namespace root\wmi -ClassName MSAcpi_ThermalZoneTemperature
```

该接口返回的温度值在实际使用中基本不变化，无法作为实时调速依据，仅可用于存在性验证。

---

## ACPI / DSDT 分析过程

### 1. 导出 ACPI 表

使用 `acpidump` 获取系统 ACPI 数据：

```powershell
acpidump.exe -b
```

生成的 `DSDT.dat` / `SSDT*.dat` 文件可用于反编译分析。

---

### 2. 反编译 DSDT

```powershell
iasl.exe -d dsdt.dat
```

得到 `dsdt.dsl`，用于后续搜索。

---

### 3. 定位 WMI 设备

在 DSDT 中搜索标准 WMI HID：

```text
PNP0C14
```

发现多个 WMI Device，其中一个设备标识为厂商相关实现。

---

### 4. 关键设备：EWMI

```asl
Device (EWMI)
{
    Name (_HID, "PNP0C14")
    Name (_UID, "EmdWmi")
    Name (_WDG, Buffer (...))
    ...
}
```

该设备通过 `_WDG` 描述向 Windows 注册 WMI 接口，供厂商驱动调用。

---

## 风扇控制接口

在 Windows 下可见对应的 WMI 类：

```text
root\wmi: EmdAcpi_FanControl
```

其方法：

```text
FanControl(Data)
```

### 参数含义（实测）

| Data | 含义           |
| ---- | ------------ |
| 0    | 自动 / BIOS 控制 |
| 1    | 静音 / 低速      |
| 2    | 性能 / 高速      |

该接口不返回当前转速，仅负责模式切换。

---

## 结论

* HKC 笔记本风扇控制通过 **厂商自定义 ACPI WMI 接口** 实现
* 不支持连续调速，仅支持 **预设档位切换**
* 可被 PowerShell 直接调用，无需安装额外驱动
* 适合作为 FanControl / NBFC 的外部命令或参考实现

---

## 风险提示

* 不同型号 HKC 笔记本实现可能不同
* 错误使用高转速模式可能增加噪音与功耗
* 所有操作风险自负

---

## License

MIT


