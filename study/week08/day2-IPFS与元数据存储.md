# Week 8 - Day 2: IPFS与元数据存储

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解IPFS工作原理
- ✅ 使用Pinata管理文件
- ✅ 掌握元数据上传流程
- ✅ 前端集成IPFS
- ✅ 理解Arweave永久存储

---

## Part 1: IPFS基础 (1.5小时)

### 1.1 IPFS原理

IPFS (InterPlanetary File System) 是一个点对点的分布式文件系统。

**核心概念**：
- **CID (Content Identifier)**: 基于内容的寻址，文件内容改变CID也会改变。
- **Pinning**: 保证文件不被垃圾回收机制清除。
- **Gateway**: 通过HTTP访问IPFS内容。

### 1.2 使用IPFS CLI

```bash
# 初始化
ipfs init

# 添加文件
ipfs add metadata.json
# 输出: QmX... metadata.json

# 访问文件
ipfs cat QmX...

# 启动守护进程
ipfs daemon
```

---

## Part 2: Pinata API集成 (2小时)

Pinata是最流行的IPFS Pinning服务，提供方便的API。

### 2.1 上传文件

```javascript
// utils/pinata.js
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

const JWT = 'YOUR_PINATA_JWT';

export const uploadFileToIPFS = async (file) => {
  try {
    const formData = new FormData();
    formData.append('file', file);
    
    const metadata = JSON.stringify({
      name: 'My NFT Image',
    });
    formData.append('pinataMetadata', metadata);
    
    const options = JSON.stringify({
      cidVersion: 0,
    });
    formData.append('pinataOptions', options);

    const res = await axios.post(
      "https://api.pinata.cloud/pinning/pinFileToIPFS",
      formData,
      {
        maxBodyLength: "Infinity",
        headers: {
          'Content-Type': `multipart/form-data; boundary=${formData._boundary}`,
          Authorization: `Bearer ${JWT}`
        }
      }
    );
    
    return res.data.IpfsHash;
  } catch (error) {
    console.error('Error uploading file to IPFS:', error);
    throw error;
  }
};
```

### 2.2 上传JSON元数据

```javascript
export const uploadJSONToIPFS = async (jsonData) => {
  try {
    const data = JSON.stringify({
      pinataContent: jsonData,
      pinataMetadata: {
        name: `${jsonData.name} Metadata`
      }
    });

    const res = await axios.post(
      "https://api.pinata.cloud/pinning/pinJSONToIPFS",
      data,
      {
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${JWT}`
        }
      }
    );
    
    return res.data.IpfsHash;
  } catch (error) {
    console.error('Error uploading JSON to IPFS:', error);
    throw error;
  }
};
```

---

## Part 3: 前端IPFS集成 (1.5小时)

### 3.1 图片上传组件

```javascript
// components/ImageUpload.jsx
import React, { useState } from 'react';
import { uploadFileToIPFS } from '../utils/pinata';

const ImageUpload = ({ onUpload }) => {
  const [file, setFile] = useState(null);
  const [preview, setPreview] = useState('');
  const [uploading, setUploading] = useState(false);

  const handleFileChange = (e) => {
    const selectedFile = e.target.files[0];
    setFile(selectedFile);
    setPreview(URL.createObjectURL(selectedFile));
  };

  const handleUpload = async () => {
    if (!file) return;
    
    setUploading(true);
    try {
      const cid = await uploadFileToIPFS(file);
      const ipfsUrl = `ipfs://${cid}`;
      onUpload(ipfsUrl);
    } catch (error) {
      alert('Upload failed');
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="upload-container">
      <input type="file" onChange={handleFileChange} accept="image/*" />
      
      {preview && (
        <div className="preview">
          <img src={preview} alt="Preview" style={{ maxWidth: '200px' }} />
        </div>
      )}
      
      <button onClick={handleUpload} disabled={uploading || !file}>
        {uploading ? 'Uploading...' : 'Upload to IPFS'}
      </button>
    </div>
  );
};
```

### 3.2 IPFS图片显示

```javascript
// components/IPFSImage.jsx
import React from 'react';

const GATEWAY = 'https://gateway.pinata.cloud/ipfs/';

const IPFSImage = ({ src, alt, ...props }) => {
  const getGatewayUrl = (ipfsUrl) => {
    if (!ipfsUrl) return '';
    if (ipfsUrl.startsWith('http')) return ipfsUrl;
    return ipfsUrl.replace('ipfs://', GATEWAY);
  };

  return (
    <img 
      src={getGatewayUrl(src)} 
      alt={alt} 
      onError={(e) => {
        e.target.src = '/placeholder.png'; // 备用图片
      }}
      {...props} 
    />
  );
};
```

---

## Part 4: Arweave集成 (1小时)

Arweave提供永久存储，适合高价值NFT。

### 4.1 Arweave上传

```javascript
import Arweave from 'arweave';

const arweave = Arweave.init({
  host: 'arweave.net',
  port: 443,
  protocol: 'https'
});

export const uploadToArweave = async (data, wallet) => {
  try {
    const transaction = await arweave.createTransaction({
      data: data
    }, wallet);

    transaction.addTag('Content-Type', 'application/json');

    await arweave.transactions.sign(transaction, wallet);
    const response = await arweave.transactions.post(transaction);

    if (response.status === 200) {
      return `https://arweave.net/${transaction.id}`;
    } else {
      throw new Error('Upload failed');
    }
  } catch (error) {
    console.error('Arweave error:', error);
    throw error;
  }
};
```

---

## 📝 今日作业

### 作业1: 开发批量上传工具

编写一个Node.js脚本：
1. 读取本地 `assets` 目录下的所有图片
2. 批量上传到Pinata
3. 生成对应的 metadata JSON 文件
4. 批量上传 metadata JSON
5. 导出所有Token URI的列表

### 作业2: 实现NFT铸造页面

前端实现：
1. 图片上传区域
2. 属性填写表单 (Traits)
3. 自动生成Metadata并上传
4. 调用合约的 `mint` 函数

### 作业3: IPFS网关测速

编写一个小工具：
1. 测试不同IPFS Gateway的加载速度
2. 选择最快的Gateway来显示图片
3. 实现加载失败自动切换Gateway

---

## ✅ 检查清单

- [ ] 注册并配置Pinata API
- [ ] 能够通过代码上传文件和JSON
- [ ] 前端能正确解析并显示 `ipfs://` 链接
- [ ] 理解Arweave与IPFS的区别
- [ ] 掌握元数据生成的自动化流程

---

## 📅 明日预告

明天学习Marketplace合约开发：
- 市场合约架构设计
- 上架、购买、取消功能
- 报价(Offer)系统
- 交易费与版税分发

**🎉 完成Day 2！你的数据现在已去中心化存储！**
