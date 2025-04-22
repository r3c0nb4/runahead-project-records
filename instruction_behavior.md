# Test instructions in Runahead

## 1. Main purpose and experiment design
We observe the behavior of ARMv8 instructions during runahead execution. By leveraging a Spectre v1 gadget to open a speculative window, we monitor the execution phenomena of individual ARM instructions within this window: whether they execute normally, get discarded, or exhibit anomalous behaviors.

```
if(index < array_size){
    /*
    * ex. arm_ops(index, add)
    */
    arm_ops(index, ops);
    tmp = reloadbuffer[buffer[index] << 12];
}
```

## 2. General arithmetic operations
### 2.1 ADD: Executed normally
```
asm volatile("mov x2, #1\n\r":::"x2");
	if(index < size){	
		asm volatile(
			"add %[index], %[index], x2\n\r"
			:[index] "+r" (index)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index] << 12];
	}

```
### 2.2 SUB: Executed normally
```
	asm volatile(
		"add %[index],%[index], #1\n\r"
		"mov x2, #1\n\r"
		:[index] "+r" (index)
	);
	if(index < size){	
		asm volatile(
			"sub %[index], %[index], x2\n\r"
			:[index] "+r" (index)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index] << 12];
	}
```

### 2.3 MUL: executed normally
```
	uint64_t index2 = index;
	asm volatile("mov x2, #1\n\r");
	/*
	 * Spectre v1 runahead
	 */
	if(index < size){	
		asm volatile(
			"mul %[index], %[index], x2\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

```

### 2.4 UDIV: executed normally
```
	uint64_t index2 = index;
	asm volatile("mov x2, #1\n\r");
	/*
	 * Spectre v1 runahead
	 */
	if(index < size){	
		asm volatile(
			"udiv %[index], %[index], x2\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

```

## 3. Logic operations
### 3.1 EOR: executed normally
	asm volatile(
		"mov x2, #1\n\r"
		:::"x2"
	);
	isb();
	/*
	 * Spectre v1 runahead
	 */
	if(index < size){	
		asm volatile(
			"eor %[index], %[index], x2\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

### 3.2 AND: executed normally

Output:

```
	if(index < size){	
		asm volatile(
			"and %[index], %[index], %[index]\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}
```

### 3.3 LSL, LSR executed normally
```
	asm volatile(
		"mov x2, #1\n\r"
		:::"x2"
	);
	isb();
	/*
	 * Spectre v1 runahead
	 */
	if(index < size){	
		asm volatile(
			"LSL %[index], %[index], x2\n\r"
			"LSR %[index], %[index], x2\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

```
### 3.4 MVN: executed normally
```
	if(index < size){	
		asm volatile(
			"mvn %[index], %[index]\n\r"
			"mvn %[index], %[index]\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

```

### 3.5 ORR: executed normally
```
	if(index < size){	
		asm volatile(
			"orr %[index], %[index], x2\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}
```

## 4. Bit manipulation 
### 4.1 BFI: executed normally
```
	if(index < size){	
		asm volatile(
			"mov x2, #1\n\r"
			"bfi %[index], x2, #1, #1\n\r"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}
```

### 4.2 

## 5. Branch
### 5.1 B: executed normally
```
	if(index < size){	
		asm volatile(
			"b 1f\n\r"
			"add %[index], %[index], x2\n\r"
			"1:"
			:[index] "+r" (index2)
			::"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}

```

## 6. Memory access
### 6.1 LDR: executed normally
```
	if(index < size){	
		asm volatile(
			"ldr %[index], %[data]\n\r"
			:[index] "+r" (index2)
			:[data] "m" (index)
			:"x2"
		);
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}
```

### 6.2 STR: executed normally
```
	if(index < size){	
		asm volatile (
        	"str %x[input], [%[output_addr]] \n"  
        	:                                    
        	: [input] "r" (index),              
          	[output_addr] "r" (&index2)      
        	: "memory"                        
    );
		pick = reloadbuffer[fake_buffer[index2] << 12];
	}
```

