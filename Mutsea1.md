以下是 **《山海经》地图项目从建模到部署的完整工作流详解**，涵盖Blender建模优化、OAR存档管理、传送系统配置等关键步骤：

---

### **1. Blender建模与导出规范**
#### **(1) 模型制作要求**
- **比例统一**：设定1 Blender单位=1 OpenSim米（避免缩放问题）
- **材质优化**：
  - 使用单一UV贴图（避免多重纹理）
  - 纹理命名禁用中文（如 `mountain_diffuse.jpg`）
- **LOD层级**（细节分级）：
  ```python
  # Blender Python脚本示例：生成LOD
  bpy.ops.object.generate_lod(levels=3, ratio=0.5)
  ```

#### **(2) 导出DAE流程**
1. 选择模型 → 文件 → 导出 → Collada (.dae)
2. **关键参数**：
   - ☑ 仅导出选中物体
   - ☑ 应用变换（Apply Transforms）
   - ☐ 禁用三角化（保留四边形优化）
   - ☑ 包含UV和法线

#### **(3) 模型分类存储**
```
/shanhaijing_assets
   /zhongshan
      jianmu_tree.dae
      kunlun_mountain.dae
   /nanhai
      coral_palace.dae
      dragon_statue.dae
```

---

### **2. 本地测试与OAR打包**
#### **(1) 临时Region搭建**
```bash
# 在OpenSim服务器创建测试区域
create region TestRegion 1000 1000
```
- **地形预加载**：
  ```bash
  terrain load test_terrain.raw
  ```

#### **(2) Firestorm批量上传**
1. **快捷键操作**：
   - `Ctrl+3` 打开建筑工具
   - `Ctrl+U` 批量上传模型
2. **权限设置**：
   - 勾选"Next Owner可以复制/修改"
   - 设置"价格为0"（避免交易问题）

#### **(3) 布局调整技巧**
- **坐标对齐**：
  ```lsl
  // 用LSL脚本精确摆放
  llSetPos(<128.0, 128.0, 25.5>);
  llSetRot(llEuler2Rot(<0, 0, 45> * DEG_TO_RAD));
  ```
- **物理碰撞测试**：
  - 按`Ctrl+Alt+P`开启物理模拟
  - 检查角色是否卡入模型

#### **(4) 导出OAR存档**
```bash
# 在服务器控制台执行
save oar shanhaijing_temp.oar -m "Initial version"
```
**参数说明**：
- `-m` 添加备注
- `--assets` 包含所有依赖资源

---

### **3. 生产环境部署**
#### **(1) 区域初始化**
```bash
# 创建六大区域
create region ZhongShan 1000 1000
create region NanShan 1000 1256
create region YunTing 1256 1256
...（其他区域同理）
```

#### **(2) OAR分区域导入**
```bash
# 中山区域加载
change region ZhongShan
load oar zhongshan.oar --merge --force-terrain

# 南海区域特殊设置（水下效果）
set region_environment NanHai --water-level 20 --underwater-fog 0.7
```
**关键参数**：
- `--merge` 保留现有用户建筑
- `--force-terrain` 强制覆盖地形

#### **(3) 数据库优化（MySQL配置）**
```ini
# Database.ini
[Storage]
ConnectionString = "Server=localhost;Database=opensim;Uid=root;Pwd=123456;Pooling=true;"
AssetCacheSize = 2048  # MB
```

---

### **4. 传送系统开发**
#### **(1) 神兽传送门脚本**
```lsl
// 放置在南山区域的朱雀雕像
default
{
    touch_start(integer num)
    {
        key user = llDetectedKey(0);
        string region = "ZhongShan"; 
        vector landing_point = <128, 128, 50>;
        vector look_at = <1, 0, 0>;
        
        // 播放传送特效
        llParticleSystem([
            PSYS_PART_FLAGS, PSYS_PART_EMISSIVE_MASK,
            PSYS_SRC_TEXTURE, "fire_texture"
        ]);
        
        // 延迟0.5秒后传送
        llSleep(0.5);
        llTeleportAgent(user, region, landing_point, look_at);
    }
}
```

#### **(2) 跨区域HUD导航**
```lsl
// HUD核心代码片段
showRegionMap()
{
    llDialog(llGetOwner(),
        "《山海经》六大仙域：",
        ["中山", "南山", "南海", "云庭", "东海", "中海"], 
        channel);
}

listen(integer chan, string name, key id, string msg)
{
    if (msg == "中山") 
        llTeleportAgent(id, "ZhongShan", <128,128,25>, <0,0,0>);
    else if (msg == "南山")
        ...（其他区域同理）
}
```

---

### **5. NPC与交互系统**
#### **(1) 会说话的瑞兽**
```lsl
// 昆仑山白泽神兽对话
string[] lore_msgs = [
    "吾乃白泽，通晓万物之情...",
    "中山之东有青丘，九尾狐居焉..."
];

default
{
    touch_start(integer num)
    {
        integer idx = llFloor(llFrand(llGetListLength(lore_msgs)));
        llSay(0, lore_msgs[idx]);
        
        // 同步播放嘴部动画
        llStartAnimation("mouth_open");
    }
}
```

#### **(2) 动态天气系统**
```lsl
// 云庭区域天气控制器
integer is_raining = FALSE;

toggleWeather()
{
    is_raining = !is_raining;
    llRegionSetEnvironment(ENV_RAIN, is_raining);
    llOwnerSay("天气已切换：" + (is_raining ? "暴雨" : "晴空"));
}
```

---

### **6. 维护与监控**
#### **(1) 常用管理命令**
```bash
# 查看区域资源占用
show region resource ZhongShan

# 强制回收内存
gc compact

# 备份单个区域
save oar zhongshan_backup.oar --scope Region --assets
```

#### **(2) 日志分析技巧**
```bash
# 监控脚本错误
tail -f logs/opensim.log | grep "LSL ERROR"

# 统计活跃用户
show users --last 30m
```

---

### **故障排除清单**
| 问题现象                  | 解决方案                          |
|---------------------------|-----------------------------------|
| 模型贴图丢失              | 检查纹理路径是否含中文/特殊字符   |
| 传送后角色卡在地下        | 调整目标点Z坐标+5米              |
| OAR导入后脚本不运行       | 用`reset script *`命令重置       |
| 多人同时登录导致崩溃      | 增加`[Scene]MaxAgents = 50`参数  |

通过这套流程，你可以在2-3周内完成《山海经》世界的完整部署。建议先在一个测试区域验证所有系统，再批量推广到六大区域。
