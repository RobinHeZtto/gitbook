# 外观模式

&#x20;       外观模式（Facade Pattern）要求一个子系统的外部与内部的通信通过一个**统一的对象**进行，使得子系统的使用更加方便简单。在开发过程中的运用频率非常高，尤其在各种SDK中使用非常多，通过某一外观类使得**整个系统只有一个统一的高层接口**，以达到**够降低用户的使用成本，屏蔽实现细节**的目的。

![](<../../../.gitbook/assets/image (444).png>)

* **Facade**：这个外观类为子系统中Packages 1、2、3提供一个共同的对外接口
* **Clients**：客户对象通过一个外观接口读写子系统中各接口的数据资源。
* **Packages**：客户可以通过外观接口读取的内部库。

```
/* Complex parts */

class CPU {
	public void freeze() { ... }
	public void jump(long position) { ... }
	public void execute() { ... }
}

class Memory {
	public void load(long position, byte[] data) {
		...
	}
}

class HardDrive {
	public byte[] read(long lba, int size) {
		...
	}
}

/* Façade */

class Computer {
	public void startComputer() {
		cpu.freeze();
		memory.load(BOOT_ADDRESS, hardDrive.read(BOOT_SECTOR, SECTOR_SIZE));
		cpu.jump(BOOT_ADDRESS);
		cpu.execute();
	}
}

/* Client */

class Client {
	public static void main(String[] args) {
		Computer facade = new Computer();
		facade.startComputer();
	}
}
```
