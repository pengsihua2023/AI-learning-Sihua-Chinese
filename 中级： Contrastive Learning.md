## 对比学习（Contrastive Learning）
- 重要性：
对比学习是一种自监督学习方法，通过让模型学习区分“相似”和“不相似”的数据对，提取高质量的特征表示。  
它是现代无监督学习的核心，驱动了如 SimCLR、MoCo（计算机视觉）和 CLIP（多模态学习）的成功。  
在标注数据稀缺的场景下（如医疗影像、稀有语言），对比学习能显著减少对标注的依赖。  
- 核心概念：
对比学习的目标是让相似的数据对（正样本对）在特征空间中靠得更近，不相似的数据对（负样本对）靠得更远。   
使用对比损失函数（如 InfoNCE 损失）来优化特征表示。 
- 比喻：像“找朋友游戏”，模型学会把“朋友”（相似图片/文本）聚在一起，把“陌生人”（不相似数据）分开。  
- 应用：
图像分类（SimCLR、MoCo）：用少量标注数据实现高精度分类。  
多模态学习（CLIP）：图文搜索、图像生成（如 DALL·E）。  
<img width="1600" height="919" alt="image" src="https://github.com/user-attachments/assets/5d389da9-c6c7-46d5-a1c5-096422a5328b" />

编写一个基于PyTorch的最简单Contrastive Learning示例，使用真实数据集（MNIST手写数字数据集），实现对比学习以学习图像特征嵌入。模型将使用SimCLR风格的对比损失（NT-Xent），目标是使相同数字的图像嵌入更接近，不同数字的嵌入更远离。结果将通过可视化嵌入空间（使用t-SNE降维）和评估k-NN分类准确率来展示。

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
from torchvision import datasets, transforms
import numpy as np
from sklearn.manifold import TSNE
from sklearn.neighbors import KNeighborsClassifier
from sklearn.metrics import accuracy_score
import matplotlib.pyplot as plt

# 定义简单卷积网络
class SimpleCNN(nn.Module):
    def __init__(self, embed_dim=128):
        super(SimpleCNN, self).__init__()
        self.encoder = nn.Sequential(
            nn.Conv2d(1, 16, 3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(16, 32, 3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Flatten(),
            nn.Linear(32 * 7 * 7, embed_dim),
            nn.ReLU()
        )
    
    def forward(self, x):
        return self.encoder(x)

# 对比损失（NT-Xent）
class NTXentLoss(nn.Module):
    def __init__(self, temperature=0.5):
        super(NTXentLoss, self).__init__()
        self.temperature = temperature
        self.criterion = nn.CrossEntropyLoss()
    
    def forward(self, z1, z2, batch_size):
        # 归一化嵌入
        z1 = nn.functional.normalize(z1, dim=1)
        z2 = nn.functional.normalize(z2, dim=1)
        z = torch.cat([z1, z2], dim=0)
        
        # 计算相似性矩阵
        sim_matrix = torch.mm(z, z.transpose(0, 1)) / self.temperature
        labels = torch.cat([torch.arange(batch_size) for _ in range(2)], dim=0)
        labels = (labels.unsqueeze(0) == labels.unsqueeze(1)).float()
        labels = labels.to(z.device)
        
        # 掩码去除自身相似性
        mask = torch.eye(labels.shape[0], dtype=torch.bool).to(z.device)
        sim_matrix = sim_matrix.masked_fill(mask, -9e15)
        
        # 计算损失
        loss = self.criterion(sim_matrix, labels.argmax(dim=1))
        return loss

# 数据增强
def get_data_loaders():
    transform = transforms.Compose([
        transforms.RandomResizedCrop(28, scale=(0.8, 1.0)),
        transforms.RandomRotation(10),
        transforms.ToTensor()
    ])
    train_dataset = datasets.MNIST(root='./data', train=True, download=True,
                                 transform=transforms.Compose([transforms.ToTensor()]))
    train_loader = DataLoader(train_dataset, batch_size=128, shuffle=True)
    
    # 用于评估的原始数据（无增强）
    test_dataset = datasets.MNIST(root='./data', train=False, download=True,
                                transform=transforms.ToTensor())
    test_loader = DataLoader(test_dataset, batch_size=128, shuffle=False)
    
    return train_loader, test_loader

# 训练模型
def train_model(model, train_loader, epochs=10):
    optimizer = optim.Adam(model.parameters(), lr=0.001)
    criterion = NTXentLoss(temperature=0.5)
    
    model.train()
    for epoch in range(epochs):
        total_loss = 0
        for batch_idx, (images, _) in enumerate(train_loader):
            images = images.to(device)
            # 创建两组增强视图
            aug1 = transforms.RandomResizedCrop(28, scale=(0.8, 1.0))(images)
            aug2 = transforms.RandomRotation(10)(images)
            
            optimizer.zero_grad()
            z1 = model(aug1)
            z2 = model(aug2)
            loss = criterion(z1, z2, images.size(0))
            loss.backward()
            optimizer.step()
            total_loss += loss.item()
        
        print(f'Epoch [{epoch+1}/{epochs}], Loss: {total_loss / len(train_loader):.4f}')

# 评估和可视化
def evaluate_and_visualize(model, test_loader):
    model.eval()
    embeddings = []
    labels = []
    
    with torch.no_grad():
        for images, targets in test_loader:
            images = images.to(device)
            emb = model(images)
            embeddings.append(emb.cpu().numpy())
            labels.append(targets.numpy())
    
    embeddings = np.concatenate(embeddings, axis=0)
    labels = np.concatenate(labels, axis=0)
    
    # t-SNE降维
    tsne = TSNE(n_components=2, random_state=42)
    embeddings_2d = tsne.fit_transform(embeddings[:1000])  # 取前1000个样本
    
    # 可视化嵌入
    plt.figure(figsize=(10, 8))
    for i in range(10):
        mask = labels[:1000] == i
        plt.scatter(embeddings_2d[mask, 0], embeddings_2d[mask, 1], label=f'Digit {i}', alpha=0.5)
    plt.legend()
    plt.title('t-SNE Visualization of MNIST Embeddings')
    plt.savefig('mnist_embeddings.png')
    plt.close()
    print("t-SNE visualization saved as 'mnist_embeddings.png'")
    
    # k-NN评估
    knn = KNeighborsClassifier(n_neighbors=5)
    knn.fit(embeddings[:8000], labels[:8000])
    pred = knn.predict(embeddings[8000:])
    accuracy = accuracy_score(labels[8000:], pred)
    print(f'k-NN Classification Accuracy: {accuracy:.4f}')

def main():
    global device
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = SimpleCNN(embed_dim=128).to(device)
    train_loader, test_loader = get_data_loaders()
    train_model(model, train_loader, epochs=10)
    evaluate_and_visualize(model, test_loader)

if __name__ == "__main__":
    main()
```

### 代码说明：
1. **数据集**：
   - 使用MNIST手写数字数据集（60,000训练样本，10,000测试样本）。
   - 训练时应用数据增强（随机裁剪和旋转），生成两组视图以进行对比学习。
   - 测试时使用原始图像评估嵌入质量。

2. **模型结构**：
   - 简单CNN编码器：两层卷积（带ReLU和MaxPool），展平后接全连接层，输出128维嵌入。
   - 对比损失（NT-Xent）：使相同样本的增强视图嵌入接近，不同样本的嵌入远离，温度参数设为0.5。

3. **训练**：
   - 使用Adam优化器，学习率0.001，训练10个epoch。
   - 每批次对两组增强视图计算对比损失，优化嵌入空间。

4. **评估与可视化**：
   - **可视化**：对测试集前1000个样本的嵌入进行t-SNE降维，绘制二维散点图，按数字类别着色，保存为`mnist_embeddings.png`。
   - **评估**：使用k-NN分类器（k=5）在嵌入空间上评估分类准确率，训练集为前8000个样本，测试集为剩余样本。
   - 输出k-NN分类准确率和可视化文件路径。

5. **依赖**：
   - 需安装`torch`、`torchvision`、`sklearn`、`matplotlib`（`pip install torch torchvision scikit-learn matplotlib`）。
   - MNIST数据集会自动下载到`./data`目录。

### 运行结果：
- 输出每轮训练的平均损失。
- 生成`mnist_embeddings.png`，展示嵌入空间中不同数字类别的分布（理想情况下同类数字聚类，异类分开）。
- 输出k-NN分类准确率，反映嵌入空间的质量。
- 运行时间较长（因t-SNE和k-NN计算），可在CPU上运行，但GPU会更快。

### 注意：
- 散点图保存在运行目录下，可用图像查看器检查，颜色表示不同数字类别。
- 模型简单，适合展示对比学习概念；实际应用可增加网络深度或调整超参数。
