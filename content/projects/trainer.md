+++
author = "gio"
title = "A golang trainer"
date = "2019-03-09"
description = "A golang program that interact with Windows API to modify memory of another program"
tags = [
    "golang",
    "windows"
]
+++

A golang program that interact with Windows API to modify memory of another program
<!--more-->
---
<!-- Your front matter up here -->

## Introduction

To get more experience with windows internal and golang, I wrote this tool capable of passing the Cheat Engine tutorial. I'm writing this blog post to share with you what I have learned and hope to teach you something.

## Get processes
{{< highlight go "linenos=table,linenostart=42" >}}
func processes() ([]WindowsProcess, error) {
	handle, err := windows.CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
	if err != nil {
		return nil, err
	}
	defer windows.CloseHandle(handle)

	var entry windows.ProcessEntry32
	entry.Size = uint32(unsafe.Sizeof(entry))
	// get the first process
	err = windows.Process32First(handle, &entry)
	if err != nil {
		return nil, err
	}

	results := make([]WindowsProcess, 0, 50)
	for {
		results = append(results, newWindowsProcess(&entry))

		err = windows.Process32Next(handle, &entry)
		if err != nil {
			// windows sends ERROR_NO_MORE_FILES on last process
			if err == syscall.ERROR_NO_MORE_FILES {
				return results, nil
			}
			return nil, err
		}
	}
}
{{< / highlight >}}


{{< notice note >}}

[CreateToolhelp32Snapshot](https://docs.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-createtoolhelp32snapshot)

Takes a snapshot of the specified processes, as well as the heaps, modules, and threads used by these processes.
{{< /notice >}}

## Open process
{{< highlight go "linenos=table,linenostart=101" >}}
func main() {	
	list, err := processes()
	proc := findProcessByName(list, "Tutorial-x86_64.exe")
	pid := proc.ProcessID
	PROCESS_ALL_ACCESS := syscall.STANDARD_RIGHTS_REQUIRED | syscall.SYNCHRONIZE | 0xfff
	handle, err = w32.OpenProcess(PROCESS_ALL_ACCESS, true, ptr(pid))
	if err != nil {
		fmt.Println(err)
	}
	module, _ := EnumProcessModule(windows.Handle(handle))

	ret, _, err := getModuleInformation.Call(uintptr(handle), uintptr(module),
		uintptr(unsafe.Pointer(&info)), unsafe.Sizeof(info))
		{{< / highlight >}}
		
## Helper functions

### write
{{< highlight go "linenos=table,linenostart=139" >}}
func writeHelper(address int, byteArray []byte) {
	w32.WriteProcessMemory(handle, uintptr(address), uintptr(unsafe.Pointer(&byteArray[0])), uintptr(len(byteArray)))
}
{{< / highlight >}}

### read
{{< highlight go "linenos=table,hl_lines=167,linenostart=165" >}}
func readHelper(baseOffset int, offsets []int) int {
	baseNOffset := int(info.BaseOfDll + uintptr(baseOffset))
	val, _, _ := w32.ReadProcessMemory(handle, ptr(baseNOffset), 8)
	fstsummed := 0

	for _, k := range offsets {
		fstsummed = uint16ToByte(val) + k
		val, _, _ = w32.ReadProcessMemory(handle, ptr(fstsummed), 4)
	}
	return fstsummed
}
func simpleReadWrite(baseOffset int, offsets []int, byteArray []byte) {
	address := readHelper(baseOffset, offsets)
	writeHelper(address, byteArray)
}

{{< / highlight >}}

## step 2
{{< highlight go "linenos=table,linenostart=139" >}}
func step2() {
	baseNOffset := int(info.BaseOfDll + 0x00306A70)
	val, _, _ := w32.ReadProcessMemory(handle, ptr(baseNOffset), 4)
	add := uint16ToByte(val)
	writeHelper(add+0x7F0,[]byte{0xe8, 3, 0, 0})
}
{{< / highlight >}}

## step 3
{{< highlight go "linenos=table,linenostart=139" >}}
func step3() {
	baseNOffset := int(info.BaseOfDll + 0x00309CA0)
	val, _, _ := w32.ReadProcessMemory(handle, ptr(baseNOffset), 4)
	fstsummed := 0
	for _, k := range []int{0x58, 0x88, 0x10, 0x480, 0x20, 0x960, 0x810} {
		fstOffset := uint16ToByte(val)

		fstsummed = fstOffset + k
		val, _, _ = w32.ReadProcessMemory(handle, ptr(fstsummed), 4)
	}
	writeHelper(fstsummed,[]byte{0x88,0x13,0,0})
}
{{< / highlight >}}

## step 4
{{< highlight go "linenos=table,linenostart=139" >}}
func step4() {
	simpleReadWrite(0x00306AA0, []int{0xd0, 0x28, 0xF8, 0x18, 0x80, 0x28, 0x818}, []byte{0x00, 0x40, 0x9c, 0x45})
	simpleReadWrite(0x00306AA0, []int{0xd0, 0x28, 0xF8, 0x18, 0x88, 0x28, 0x820}, []byte{0x00, 0x00, 0x00, 0x00, 0x00, 0x88, 0xb3, 0x40})
}
{{< / highlight >}}

## step 5
{{< highlight go "linenos=table,linenostart=139" >}}
func step5() {
	writeHelper(0x10002C5B8, []byte{0x90, 0x90})
}
{{< / highlight >}}

## step 6
{{< highlight go "linenos=table,linenostart=139" >}}
func step6() {
	address := readHelper(0x00306AD0, []int{0})
	writeHelper(address, []byte{0x88, 0x13, 0, 0})
}
{{< / highlight >}}
## step 7
{{< highlight go "linenos=table,linenostart=139" >}}
func step7() {
	writeHelper(int(info.BaseOfDll+0x2d4f7), []byte{0x83, 0x86, 0xE0, 0x07, 0x00, 0x00, 0x02})
}
{{< / highlight >}}
## step 8
{{< highlight go "linenos=table,linenostart=139" >}}
func step8() {
	address := readHelper(0x00306B00, []int{0x10, 0x18, 0x0, 0x18})
	writeSet(address, []byte{0x88, 0x13})
}

{{< / highlight >}}
## step 9
{{< highlight go "linenos=table,linenostart=139" >}}

func step9() {
	ToFind := []byte{0xF3, 0x0F, 0x11, 0x43, 0x08, 0x0F}
	for i := 0; i < int(info.SizeOfImage); i++ {
		baseNOffset := int(info.BaseOfDll + uintptr(i))
		val, _, _ := w32.ReadProcessMemory(handle, ptr(baseNOffset), 6)
		b := make([]byte, 6)
		b[5] = byte(val[2] >> 8)
		b[4] = byte(val[2])
		b[3] = byte(val[1] >> 8)
		b[2] = byte(val[1])
		b[1] = byte(val[0] >> 8)
		b[0] = byte(val[0])
		res := bytes.Compare(b, ToFind)
		if res == 0 {
			fmt.Printf("found at offset %x\n\r", i)
			address := 0xFFFF0000
			w32.VirtualAllocEx(handle, uintptr(address), uintptr(2048), 0x00001000|0x00002000, 0x04)
			writeHelper(int(info.BaseOfDll+uintptr(i)), []byte{0xe9, 0x8e, 0x14, 0xfc, 0xff})
			writeHelper(address, []byte{0x83, 0x7B, 0x14, 0x01, 0x0F, 0x85, 0x0C, 0x00, 0x00, 0x00, 0xC7, 0x43, 0x08, 0x00, 0x00, 0x7A, 0x44, 0xE9, 0x5C, 0xEB, 0x03, 0x00, 0xC7, 0x43, 0x08, 0x00, 0x00, 0x00, 0x00, 0xE9, 0x50, 0xEB, 0x03, 0x00})
			oldProtect := windows.PAGE_READWRITE
			_, _, errVirtualProtectEx := procVirtualProtect.Call(uintptr(handle), uintptr(0xFFFF0000), uintptr(2048), uintptr(syscall.PAGE_EXECUTE_READWRITE), uintptr(unsafe.Pointer(&oldProtect)))
			fmt.Println(errVirtualProtectEx)
		}
	}
{{< / highlight >}}
