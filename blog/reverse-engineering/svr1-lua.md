
# SVR1逆向 - 内嵌Lua

SynthV R1内部与华老师同期的SHIRO一样，大量在程序内嵌入Lua脚本加快部分模块的开发速度。SynthV R1的多种核心业务代码内也使用了Lua，但它的Lua代码被编译过，逆向时需要自行反汇编。

## 来源

用Ghidra打开SynthV R1的Linux版二进制，找到`synv_create_frontend`函数，里面会调用`lua_load`加载Lua二进制，第二个参数是它的loader函数，进去看看：

```c
void* frontend_loader(lua_State* state, void* data, size_t *size)

{
  *size = 0x197d3;
  return luac_data;
}
```

`luac_data`是一块常量，易知长度为0x197d3字节。拷贝字节码用HxD粘出保存。字节码头部如下：

```
01 00 53 19 93 0d 0a 1a 0a 78 56 00 00 00 40 b9 43 04 04 04 04 08 01 00 16 00 01 00 00 00 00 00 00 00 00 19 01 00 00 00 0a 03 00 0b 00 00 00 05 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 44 01 00 00 00 0c ...
```

直接放入`luadec.exe`，报错提示：
```
.\luadec.exe -dis .\embedded_luacode.luac
D:\gitcode\luadec\vcproj-5.3\Debug\bin\luadec.exe: .\embedded_luacode.luac:1: unexpected symbol near '<\1>'
```

看起来文件结构不太对劲。扒出Lua源码对比 [(dumpHeader)](https://github.com/lua/lua/blob/v5.3/ldump.c#L201)：
```c
static void dumpHeader (DumpState *D) {
  DumpLiteral(LUA_SIGNATURE, D);    // .db 1bh, "Lua"
  DumpByte(LUAC_VERSION, D);        // .db 53h
  DumpByte(LUAC_FORMAT, D);         // .db 00h
  DumpLiteral(LUAC_DATA, D);        // .db 19h, 93h, 0dh, 0ah, 1ah, 0ah
  DumpByte(sizeof(int), D);         // .db 4
  DumpByte(sizeof(size_t), D);      // .db 8
  DumpByte(sizeof(Instruction), D); // .db 4
  DumpByte(sizeof(lua_Integer), D); // .db 4
  DumpByte(sizeof(lua_Number), D);  // .db 4
  DumpInteger(LUAC_INT, D);         // 0x5678
  DumpNumber(LUAC_NUM, D);          // 370.5
}
```
可见SynthV R1的Lua引擎做过点常规手脚，结合`luaU_undump`函数内容推知，编译出的Lua二进制头实际应该为：
```x86asm
.db 01h                             ;LUA_SIGNATURE
.db 00h                             ;LUAC_FORMAT
.db 53h                             ;LUAC_VERSION
.db 19h, 93h, 0dh, 0ah, 1ah, 0ah    ;LUAC_DATA
.dd 5678h                           ;LUAC_INT
.dd 43b94000h                       ;LUAC_NUM (float)370.5
.db 04h, 04h, 04h, 04h, 08h         ;lua_Integer, Instruction, lua_Number, int, size_t的尺寸，不得不说这块调的真心乱完了
```

修改Lua配置文件以达到上述尺寸要求，并修改lua的`checkHeader`函数：
```c
// SVR1 Edit:
static void checkHeader (LoadState *S) {
  //checkliteral(S, LUA_SIGNATURE + 1, "not a");  /* 1st char already checked */
  if (LoadByte(S) != LUAC_FORMAT)
    error(S, "format mismatch in");
  if (LoadByte(S) != LUAC_VERSION)
    error(S, "version mismatch in");
  checkliteral(S, LUAC_DATA, "corrupted");
  if (LoadInteger(S) != LUAC_INT)
    error(S, "endianness mismatch in");
  if (LoadNumber(S) != LUAC_NUM)
    error(S, "float format mismatch in");
  checksize(S, lua_Integer);
  checksize(S, Instruction);
  checksize(S, lua_Number);
  checksize(S, int);
  checksize(S, size_t);
}
```

重新编译luadec，产生崩溃。推测后续数据结构依然为打乱状态。

**未完**
