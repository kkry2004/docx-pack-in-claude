# docx-pack-in-claude
---
name: docx-pack
description: >
  用 node.js zlib 正确打包 .docx。禁止 PowerShell Compress-Archive（路径反斜杠、文件乱序，网站解析必挂）。

---

# DOCX 正确打包

**绝对不能用 `powershell Compress-Archive`** — 路径是反斜杠、文件顺序随机，网站解析直接报错。

只能用 node.js 手动构建 ZIP。脚本另存为 `pack.js`，放在项目目录：

```javascript
// pack.js — node pack.js ./unpacked ./output.docx
const fs = require('fs'), zlib = require('zlib'), path = require('path');
const src = process.argv[2], out = process.argv[3];

function crc32(b) { let c=-1; for(let i=0;i<b.length;i++){c^=b[i];for(let j=0;j<8;j++)c=(c>>>1)^(c&1?0xEDB88320:0)} return (c^-1)>>>0; }

let files = [];
function walk(d, base) {
    for (const e of fs.readdirSync(d, {withFileTypes: true})) {
        const rp = path.relative(base, path.join(d, e.name)).replace(/\\/g, '/');
        e.isDirectory() ? walk(path.join(d, e.name), base) : files.push({ fp: path.join(d, e.name), rp });
    }
}
walk(src, src);

// 关键：[Content_Types].xml 必须第一，_rels/.rels 第二
files.sort((a, b) =>
    a.rp === '[Content_Types].xml' ? -1 : b.rp === '[Content_Types].xml' ? 1 :
    a.rp === '_rels/.rels' ? -1 : b.rp === '_rels/.rels' ? 1 :
    a.rp.localeCompare(b.rp));

const locals = [], centrals = [];
let offset = 0;

for (const f of files) {
    const raw = fs.readFileSync(f.fp), comp = zlib.deflateRawSync(raw);
    const store = comp.length >= raw.length, data = store ? raw : comp;
    const name = Buffer.from(f.rp, 'utf8');

    // Local file header
    const lh = Buffer.alloc(30 + name.length);
    lh.writeUInt32LE(0x04034b50, 0); lh.writeUInt16LE(20, 4);
    lh.writeUInt16LE(0x0800, 6); lh.writeUInt16LE(store ? 0 : 8, 8);
    lh.writeUInt32LE(crc32(raw), 14); lh.writeUInt32LE(data.length, 18);
    lh.writeUInt32LE(raw.length, 22); lh.writeUInt16LE(name.length, 26);
    name.copy(lh, 30);

    // Central directory header
    const cd = Buffer.alloc(46 + name.length);
    cd.writeUInt32LE(0x02014b50, 0); cd.writeUInt16LE(20, 4);
    cd.writeUInt16LE(20, 6); cd.writeUInt16LE(0x0800, 8);
    cd.writeUInt16LE(store ? 0 : 8, 10); cd.writeUInt32LE(crc32(raw), 16);
    cd.writeUInt32LE(data.length, 20); cd.writeUInt32LE(raw.length, 24);
    cd.writeUInt16LE(name.length, 28); cd.writeUInt32LE(offset, 42);
    name.copy(cd, 46);

    locals.push(lh, data); centrals.push(cd);
    offset += 30 + name.length + data.length;
}

const cdBuf = Buffer.concat(centrals);
const eocd = Buffer.alloc(22);
eocd.writeUInt32LE(0x06054b50, 0);
eocd.writeUInt16LE(files.length, 8); eocd.writeUInt16LE(files.length, 10);
eocd.writeUInt32LE(cdBuf.length, 12); eocd.writeUInt32LE(offset, 16);

fs.writeFileSync(out, Buffer.concat([...locals, cdBuf, eocd]));
console.log(`OK: ${out} (${files.length} files, ${fs.statSync(out).size}B)`);
```

## 用法

```bash
mkdir unpacked && cd unpacked && unzip -o ../start.docx
# 编辑 word/document.xml 之后...
cd .. && node pack.js ./unpacked ./output.docx
```

## PowerShell 为什么不行

| 问题                 | PowerShell    | node.js         |
| -------------------- | ------------- | --------------- |
| 路径分隔             | `\`（反斜杠） | `/`（正斜杠）   |
| Content_Types 第一个 | 不确定        | 强制锁死        |
| UTF-8 flag           | 不设          | bit 11 = 0x0800 |
| 网站解析结果         | ❌ 失败        | ✅ 通过          |
