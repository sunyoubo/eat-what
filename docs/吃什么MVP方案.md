# 吃什么 MVP 方案 - 最小可行产品

## 1. MVP 目标与范围

### 1.1 核心目标
**用最小成本验证核心价值假设：**
> 用户是否愿意使用一个基于个人健康和口味偏好的智能饮食推荐工具来解决"每天吃什么"的决策难题？

### 1.2 MVP 成功标准
| 指标 | 目标值 | 测量周期 |
|------|--------|---------|
| 注册用户数 | ≥ 500 | 上线后 4 周 |
| 7 日留存率 | ≥ 30% | 上线后 4 周 |
| 日均推荐点击率 | ≥ 20% | 上线后 2 周 |
| 用户满意度 (NPS) | ≥ 20 | 上线后 4 周 |
| 核心功能使用率 | ≥ 60% | 上线后 2 周 |

### 1.3 MVP 不做的事（Out of Scope）
- ❌ 付费订阅系统
- ❌ 社交 PK 功能
- ❌ 多平台外卖比价
- ❌ 机器学习个性化优化
- ❌ 智能体脂秤集成
- ❌ 复杂的健康报告
- ❌ 飞书/微信多端支持（先聚焦微信小程序）

---

## 2. MVP 产品能力

### 2.1 核心功能清单（Must Have）

#### 2.1.1 用户注册与基础画像
**功能描述：** 用户通过微信小程序授权登录，完成基础信息采集。

**用户流程：**
```
微信小程序授权 → 获取手机号 → 填写身体数据 → 设置口味偏好 → 完成
```

**必填字段：**
| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| 性别 | 单选 | 男/女 | - |
| 年龄 | 数字 | 18-80 岁 | - |
| 身高 | 数字 | cm | - |
| 体重 | 数字 | kg | - |
| 活动水平 | 单选 | 久坐/轻度/中度/高度 | 久坐 |
| 健康目标 | 单选 | 维持/减脂/增肌 | 维持 |

**选填字段：**
| 字段 | 类型 | 说明 |
|------|------|------|
| 口味偏好 | 多选 + 排序 | 麻辣/酸甜/清淡/原味 |
| 忌口食材 | 标签选择 | 香菜/葱/蒜/辣椒/海鲜等（10 个常见选项） |
| 菜系偏好 | 多选 | 川菜/粤菜/湘菜/江浙菜/日料/韩餐（6 个主流菜系） |
| 每日预算 | 区间选择 | 15-30 元 / 30-50 元 / 50 元以上 |

**产品要点：**
- 首次填写控制在 2 分钟内完成
- 支持后续随时修改
- 身体数据用于计算 BMR/TDEE，给出明确解释

---

#### 2.1.2 智能餐食推荐
**功能描述：** 根据用户画像和当前时间，推荐适合的午餐/晚餐选项。

**推荐逻辑（简化版规则引擎）：**
```python
def recommend_meal(user, meal_type, weather):
    candidates = get_nearby_dishes(user.location, meal_type)
    
    scored_dishes = []
    for dish in candidates:
        score = 0
        
        # 1. 硬过滤：忌口和过敏
        if has_avoided_ingredients(dish, user.avoid_ingredients):
            continue
        
        # 2. 预算过滤
        if dish.price > user.budget_max:
            continue
        
        # 3. 口味匹配（辣度、菜系）
        score += taste_match_score(user, dish) * 0.3
        
        # 4. 营养适配（基于健康目标）
        score += nutrition_fit_score(user, dish) * 0.3
        
        # 5. 环境适配（天气、季节）
        score += environment_fit_score(dish, weather) * 0.2
        
        # 6. 商家评分
        score += restaurant_rating_score(dish) * 0.2
        
        scored_dishes.append((dish, score))
    
    return sort_by_score(scored_dishes)[:5]  # 返回 Top 5
```

**推荐结果展示：**
每个推荐菜品包含：
- 菜品图片 + 名称 + 商家名称
- 价格 + 距离 + 预计配送时间
- **营养卡片**（热量、蛋白质、碳水、脂肪）
- **推荐理由**（2-3 条，如"符合您的清淡口味"、"适合雨天食用"）
- "换一批"按钮

**交互设计：**
- 下拉刷新获取新推荐
- 左滑"不感兴趣"（记录反馈）
- 右滑"喜欢"（记录反馈）
- 点击查看详情并跳转外卖平台下单

---

#### 2.1.3 营养信息展示
**功能描述：** 为每道菜品提供营养估算，帮助用户做出健康选择。

**营养数据来源（MVP 阶段）：**
1. **中国食物成分表**（标准版第 6 版）- 基础食材数据
2. ** USDA FoodData Central** - 补充数据
3. **人工标注** - 针对热门外卖菜品（100 道）

**营养计算逻辑：**
```python
# 简化版菜品营养估算
def estimate_dish_nutrition(dish_name, ingredients, portions):
    base_nutrition = lookup_ingredients(ingredients, portions)
    
    # 添加烹饪油盐估算
    base_nutrition['calories'] += 45   # 5g 油
    base_nutrition['fat'] += 5
    base_nutrition['sodium'] += 200    # 0.5g 盐
    
    # 根据烹饪方式调整
    if dish.cooking_method == 'fried':
        base_nutrition['calories'] += 80
        base_nutrition['fat'] += 9
    elif dish.cooking_method == 'steamed':
        base_nutrition['calories'] -= 20
    
    return base_nutrition
```

**展示内容：**
```
┌─────────────────────────────┐
│ 🍗 宫保鸡丁                  │
│ 商家：川香阁 · 距您 1.2km    │
│ 💰 32 元  ⏱️ 35 分钟送达      │
├─────────────────────────────┤
│ 📊 营养估算                  │
│ 🔥 热量：480 kcal           │
│ 🥩 蛋白质：28g              │
│ 🍚 碳水：35g                │
│ 🧈 脂肪：22g                │
│ 🧂 钠：920mg (⚠️ 偏高)       │
├─────────────────────────────┤
│ 💡 推荐理由                  │
│ ✓ 符合您的麻辣口味偏好       │
│ ✓ 蛋白质充足，适合增肌目标   │
│ ✓ 商家评分 4.8 分            │
└─────────────────────────────┘
```

---

#### 2.1.4 用户反馈收集
**功能描述：** 收集用户对推荐的反馈，用于优化算法。

**反馈方式：**
1. **快捷反馈**（推荐列表页）：
   - 👍 喜欢
   - 👎 不喜欢（选择原因：不好吃/不健康/太贵/不想吃这个）

2. **餐后评价**（可选，下单后推送）：
   - 口味满意度（1-5 星）
   - 健康度感知（1-5 星）
   - "还会再吃吗"（是/否）
   - 上传实物照片（可选，奖励积分）

**反馈数据用途：**
- 调整用户口味偏好权重
- 识别隐性偏好（用户说不要辣但经常选微辣）
- 优化推荐算法

---

#### 2.1.5 每日饮食概览
**功能描述：** 简单展示用户今日摄入情况。

**展示内容：**
```
┌─────────────────────────────┐
│ 📅 今日饮食概览             │
│ 2026-04-03                  │
├─────────────────────────────┤
│ 已摄入                      │
│ 🔥 1280 / 2000 kcal         │
│ 🥩 65 / 150g 蛋白质         │
│ 🍚 140 / 250g 碳水          │
│ 🧈 42 / 67g 脂肪            │
├─────────────────────────────┤
│ 💧 饮水提醒                 │
│ 已喝 800ml · 还需 1700ml    │
│ [记一笔]                    │
├─────────────────────────────┤
│ 🍽️ 用餐记录                 │
│ 早餐：燕麦粥 (320kcal)      │
│ 午餐：宫保鸡丁 (480kcal)    │
│ 晚餐：未记录                │
└─────────────────────────────┘
```

**功能边界（MVP 简化）：**
- 仅展示通过小程序推荐的餐食自动记录
- 支持手动添加其他饮食（简化输入：搜索菜品名 + 份量）
- 不提供复杂的营养分析图表

---

### 2.2 辅助功能（Nice to Have）

#### 2.2.1 天气集成
- 接入和风天气 API（免费版）
- 根据温度/天气状况给出简单建议标签
  - >30℃：推荐"清爽"、"凉拌"
  - <10℃：推荐"温热"、"炖煮"
  - 雨天：推荐"温暖舒适"

#### 2.2.2 季节性推荐
- 硬编码四季推荐规则
  - 春季：清淡养肝、时令蔬菜
  - 夏季：清热解暑、多汤水
  - 秋季：润燥滋阴
  - 冬季：温补暖身

#### 2.2.3 简单的成就系统
- 连续打卡 7 天 → "自律达人"徽章
- 累计记录 20 餐 → "美食家"徽章
- 一周平均热量达标 → "健康卫士"徽章

---

## 3. MVP 技术实现

### 3.1 技术栈选型（精简版）

| 层级 | 技术 | 说明 |
|------|------|------|
| **前端** | 微信小程序原生 | 快速上线，不跨端 |
| **UI 组件** | Vant Weapp | 轻量级组件库 |
| **后端框架** | NestJS (Node.js) | 统一语言，快速开发 |
| **数据库** | MySQL 8.0 | 关系型数据 |
| **缓存** | Redis | 会话、热点数据 |
| **对象存储** | 阿里云 OSS | 菜品图片 |
| **天气 API** | 和风天气 | 免费版够用 |
| **部署** | 阿里云 ECS + Docker | 单机部署，降低成本 |

**为什么不用 Taro 跨端？**
- MVP 阶段聚焦验证核心价值
- 微信小程序覆盖目标用户群
- 减少技术复杂度，快速迭代

---

### 3.2 系统架构（简化版）

```
┌──────────────────┐
│  微信小程序      │
│  (Vant Weapp)    │
└────────┬─────────┘
         │ HTTPS
         ▼
┌──────────────────┐
│  Nginx           │
│  (反向代理+SSL)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  NestJS Backend  │
│  (单体应用)      │
│  ┌────────────┐  │
│  │ 用户模块   │  │
│  │ 推荐模块   │  │
│  │ 菜品模块   │  │
│  │ 反馈模块   │  │
│  └────────────┘  │
└────────┬─────────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
┌───────┐ ┌───────┐ ┌────────┐
│ MySQL │ │ Redis │ │  OSS   │
└───────┘ └───────┘ └────────┘
```

**架构特点：**
- 单体应用，降低运维复杂度
- 模块化设计，便于后续拆分
- 所有服务部署在单台 ECS（4C8G）

---

### 3.3 数据库设计（MVP 精简版）

#### 3.3.1 核心表结构

```sql
-- 用户表（简化）
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    openid VARCHAR(64) UNIQUE NOT NULL COMMENT '微信小程序 openid',
    phone VARCHAR(20) COMMENT '手机号',
    nickname VARCHAR(50),
    avatar_url VARCHAR(255),
    
    -- 身体数据
    gender ENUM('male', 'female') NOT NULL,
    birth_year INT NOT NULL,
    height_cm DECIMAL(5,2) NOT NULL,
    weight_kg DECIMAL(5,2) NOT NULL,
    activity_level ENUM('sedentary', 'light', 'moderate', 'active') DEFAULT 'sedentary',
    health_goal ENUM('maintain', 'lose_weight', 'gain_muscle') DEFAULT 'maintain',
    
    -- 计算字段
    bmr DECIMAL(8,2),
    tdee DECIMAL(8,2),
    bmi DECIMAL(5,2),
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_openid (openid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户偏好表（简化）
CREATE TABLE user_preferences (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    
    -- 口味倾向
    taste_spicy TINYINT DEFAULT 3,
    taste_sweet TINYINT DEFAULT 3,
    taste_light TINYINT DEFAULT 3,
    
    -- 忌口（JSON 数组）
    avoid_ingredients JSON COMMENT '["香菜", "葱"]',
    
    -- 菜系偏好（JSON 数组）
    favorite_cuisines JSON COMMENT '["sichuan", "cantonese"]',
    
    -- 预算
    budget_min DECIMAL(8,2) DEFAULT 15,
    budget_max DECIMAL(8,2) DEFAULT 50,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 菜品表（MVP 简化）
CREATE TABLE dishes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    
    -- 分类
    cuisine_type VARCHAR(50) COMMENT '菜系',
    meal_type ENUM('breakfast', 'lunch', 'dinner'),
    
    -- 营养信息（估算）
    nutrition JSON COMMENT '{
        "calories": 450,
        "protein_g": 25,
        "carbs_g": 40,
        "fat_g": 18,
        "sodium_mg": 900
    }',
    
    -- 口味标签
    spice_level TINYINT DEFAULT 0,
    taste_tags JSON COMMENT '["spicy", "sweet"]',
    
    -- 食材（用于忌口过滤）
    ingredients JSON COMMENT '["鸡肉", "辣椒", "花生"]',
    
    -- 价格
    price_avg DECIMAL(8,2) COMMENT '平均价格',
    
    -- 关联外卖平台（MVP 阶段手动维护）
    platform ENUM('meituan', 'eleme'),
    external_id VARCHAR(64),
    poi_id VARCHAR(32),
    
    image_url VARCHAR(255),
    
    is_available BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_cuisine (cuisine_type),
    INDEX idx_meal_type (meal_type),
    FULLTEXT INDEX ft_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 商家表（MVP 简化）
CREATE TABLE restaurants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    platform ENUM('meituan', 'eleme') NOT NULL,
    poi_id VARCHAR(32) UNIQUE NOT NULL,
    
    rating DECIMAL(3,2) DEFAULT 0,
    min_price DECIMAL(8,2),
    delivery_fee DECIMAL(8,2),
    delivery_time_min INT,
    
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    address VARCHAR(255),
    
    cuisine_types JSON,
    image_url VARCHAR(255),
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_platform_poi (platform, poi_id),
    INDEX idx_location (latitude, longitude)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 推荐记录表（用于分析和优化）
CREATE TABLE recommendation_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    request_id VARCHAR(64) UNIQUE,
    
    meal_type ENUM('breakfast', 'lunch', 'dinner'),
    
    -- 上下文快照
    weather_temp DECIMAL(5,2),
    weather_condition VARCHAR(50),
    season VARCHAR(20),
    
    -- 推荐的菜品（JSON）
    recommended_dishes JSON COMMENT '[{
        "dish_id": 123,
        "score": 0.85,
        "rank": 1
    }]',
    
    -- 用户行为
    clicked_dish_id BIGINT,
    ordered_dish_id BIGINT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_time (user_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户反馈表
CREATE TABLE user_feedbacks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    dish_id BIGINT NOT NULL,
    recommendation_id BIGINT,
    
    liked BOOLEAN,
    dislike_reason ENUM('not_tasty', 'unhealthy', 'too_expensive', 'other'),
    
    taste_rating TINYINT,
    comment TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (dish_id) REFERENCES dishes(id),
    INDEX idx_user_dish (user_id, dish_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 饮食记录表
CREATE TABLE food_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    dish_id BIGINT,
    
    meal_type ENUM('breakfast', 'lunch', 'dinner', 'snack'),
    
    -- 如果是推荐的菜品，直接关联
    from_recommendation BOOLEAN DEFAULT false,
    recommendation_id BIGINT,
    
    -- 如果是手动添加
    custom_food_name VARCHAR(100),
    custom_nutrition JSON,
    
    calories DECIMAL(8,2),
    protein_g DECIMAL(8,2),
    carbs_g DECIMAL(8,2),
    fat_g DECIMAL(8,2),
    
    log_time DATETIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_date (user_id, log_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

### 3.4 核心 API 接口（MVP）

#### 3.4.1 认证与用户

```typescript
// POST /api/v1/auth/wechat
// 微信小程序登录
Request: { code: string }  // 微信登录 code
Response: {
  token: string,
  user: {
    id: number,
    nickname: string,
    avatar_url: string,
    is_new_user: boolean  // 是否需要完善资料
  }
}

// GET /api/v1/users/profile
// 获取用户资料
Response: {
  gender: string,
  birth_year: number,
  height_cm: number,
  weight_kg: number,
  activity_level: string,
  health_goal: string,
  bmr: number,
  tdee: number,
  bmi: number
}

// PATCH /api/v1/users/profile
// 更新用户资料
Request: {
  gender?: string,
  birth_year?: number,
  height_cm?: number,
  weight_kg?: number,
  activity_level?: string,
  health_goal?: string
}

// GET /api/v1/users/preferences
// 获取用户偏好
Response: {
  taste_spicy: number,
  taste_sweet: number,
  taste_light: number,
  avoid_ingredients: string[],
  favorite_cuisines: string[],
  budget_min: number,
  budget_max: number
}

// PATCH /api/v1/users/preferences
// 更新用户偏好
Request: {
  taste_spicy?: number,
  avoid_ingredients?: string[],
  favorite_cuisines?: string[],
  budget_max?: number
}
```

#### 3.4.2 推荐接口

```typescript
// POST /api/v1/recommend/meals
// 获取餐食推荐
Request: {
  meal_type: 'breakfast' | 'lunch' | 'dinner',
  latitude?: number,
  longitude?: number
}
Response: {
  request_id: string,
  recommendations: [{
    dish_id: number,
    dish_name: string,
    restaurant_name: string,
    price: number,
    distance_km: number,
    delivery_time_min: number,
    nutrition: {
      calories: number,
      protein_g: number,
      carbs_g: number,
      fat_g: number,
      sodium_mg: number
    },
    match_score: number,
    match_reasons: string[],
    image_url: string,
    platform_order_url: string  // 跳转外卖平台的链接
  }],
  daily_summary: {
    consumed_calories: number,
    target_calories: number,
    remaining_calories: number
  }
}

// POST /api/v1/recommend/feedback
// 提交推荐反馈
Request: {
  request_id: string,
  dish_id: number,
  action: 'like' | 'dislike',
  reason?: 'not_tasty' | 'unhealthy' | 'too_expensive' | 'other'
}
```

#### 3.4.3 饮食记录

```typescript
// GET /api/v1/food-logs/today
// 获取今日饮食记录
Response: {
  date: string,
  total_calories: number,
  total_protein: number,
  total_carbs: number,
  total_fat: number,
  meals: [{
    meal_type: string,
    dish_name: string,
    calories: number,
    protein_g: number,
    carbs_g: number,
    fat_g: number,
    log_time: string
  }]
}

// POST /api/v1/food-logs
// 手动添加饮食记录
Request: {
  meal_type: string,
  food_name: string,
  calories: number,
  protein_g?: number,
  carbs_g?: number,
  fat_g?: number,
  log_time?: string  // 默认当前时间
}
```

---

### 3.5 推荐算法实现（MVP 简化版）

```typescript
// NestJS Service
@Injectable()
export class RecommendationService {
  constructor(
    private dishRepo: Repository<Dish>,
    private userRepo: Repository<User>,
    private weatherService: WeatherService,
    private logger: Logger
  ) {}

  async getRecommendations(
    userId: number,
    mealType: string,
    location?: { lat: number; lng: number }
  ): Promise<RecommendationResult> {
    const requestId = uuidv4();
    
    // 1. 获取用户画像
    const user = await this.userRepo.findOne({ 
      where: { id: userId },
      relations: ['preferences']
    });
    
    // 2. 获取环境因素
    const weather = location 
      ? await this.weatherService.getCurrent(location.lat, location.lng)
      : null;
    const season = this.getSeason();
    
    // 3. 获取候选菜品（简化：不过滤距离，后续人工维护热门菜品）
    const candidates = await this.dishRepo.find({
      where: {
        meal_type: mealType,
        is_available: true
      },
      take: 50
    });
    
    // 4. 计算匹配分数
    const scoredDishes = candidates
      .filter(dish => this.filterByAvoidIngredients(dish, user.preferences))
      .filter(dish => this.filterByBudget(dish, user.preferences))
      .map(dish => ({
        dish,
        score: this.calculateMatchScore(user, dish, weather, season),
        reasons: this.generateMatchReasons(user, dish, weather)
      }))
      .filter(item => item.score > 0.5)
      .sort((a, b) => b.score - a.score)
      .slice(0, 5);  // Top 5
    
    // 5. 记录推荐日志
    await this.logRecommendation(requestId, userId, mealType, scoredDishes, weather);
    
    // 6. 获取今日饮食概览
    const dailySummary = await this.getDailySummary(userId);
    
    return {
      request_id: requestId,
      recommendations: scoredDishes.map(item => this.formatDish(item.dish, item.reasons)),
      daily_summary: dailySummary
    };
  }

  private calculateMatchScore(
    user: User,
    dish: Dish,
    weather: Weather,
    season: string
  ): number {
    let score = 0;
    
    // 口味匹配（30%）
    const tasteScore = this.calculateTasteMatch(user, dish);
    score += tasteScore * 0.3;
    
    // 营养适配（30%）
    const nutritionScore = this.calculateNutritionFit(user, dish);
    score += nutritionScore * 0.3;
    
    // 环境适配（20%）
    const envScore = this.calculateEnvironmentFit(dish, weather, season);
    score += envScore * 0.2;
    
    // 商家评分（20%）
    const ratingScore = dish.restaurant_rating ? dish.restaurant_rating / 5 : 0.5;
    score += ratingScore * 0.2;
    
    return score;
  }

  private calculateTasteMatch(user: User, dish: Dish): number {
    let score = 0.5;  // 基础分
    
    // 辣度匹配
    const spiceDiff = Math.abs(user.preferences.taste_spicy - dish.spice_level);
    score += (1 - spiceDiff / 5) * 0.4;
    
    // 菜系偏好
    if (user.preferences.favorite_cuisines?.includes(dish.cuisine_type)) {
      score += 0.3;
    }
    
    return Math.min(1, score);
  }

  private calculateNutritionFit(user: User, dish: Dish): number {
    const nutrition = dish.nutrition as any;
    
    // 根据健康目标调整评分
    if (user.health_goal === 'lose_weight') {
      // 减脂：偏好低热量、高蛋白
      const calorieScore = nutrition.calories < 500 ? 1 : nutrition.calories < 700 ? 0.7 : 0.3;
      const proteinScore = nutrition.protein_g > 20 ? 1 : 0.6;
      return calorieScore * 0.6 + proteinScore * 0.4;
    } else if (user.health_goal === 'gain_muscle') {
      // 增肌：偏好高蛋白、适中碳水
      const proteinScore = nutrition.protein_g > 25 ? 1 : nutrition.protein_g > 15 ? 0.7 : 0.3;
      return proteinScore;
    } else {
      // 维持：均衡即可
      return 0.7;
    }
  }

  private calculateEnvironmentFit(dish: Dish, weather: Weather, season: string): number {
    let score = 0.5;
    
    if (weather && weather.temperature > 30) {
      // 夏天推荐清爽
      if (dish.taste_tags?.includes('light') || dish.taste_tags?.includes('cold')) {
        score += 0.3;
      }
    } else if (weather && weather.temperature < 10) {
      // 冬天推荐温热
      if (dish.taste_tags?.includes('hot') || dish.taste_tags?.includes('soup')) {
        score += 0.3;
      }
    }
    
    // 季节适配（简化：硬编码规则）
    if (season === 'summer' && dish.cuisine_type === 'salad') {
      score += 0.2;
    } else if (season === 'winter' && dish.taste_tags?.includes('soup')) {
      score += 0.2;
    }
    
    return Math.min(1, score);
  }

  private generateMatchReasons(user: User, dish: Dish, weather: Weather): string[] {
    const reasons: string[] = [];
    
    if (user.preferences.favorite_cuisines?.includes(dish.cuisine_type)) {
      reasons.push(`符合您喜欢的${this.getCuisineName(dish.cuisine_type)}口味`);
    }
    
    if (dish.spice_level === user.preferences.taste_spicy) {
      reasons.push('辣度刚好符合您的偏好');
    }
    
    if (user.health_goal === 'lose_weight' && (dish.nutrition as any).calories < 500) {
      reasons.push('低卡路里，适合减脂');
    } else if (user.health_goal === 'gain_muscle' && (dish.nutrition as any).protein_g > 25) {
      reasons.push('高蛋白，助力增肌');
    }
    
    if (weather && weather.temperature > 30) {
      reasons.push('清爽适合炎热天气');
    } else if (weather && weather.description?.includes('雨')) {
      reasons.push('温暖舒适适合雨天');
    }
    
    return reasons.slice(0, 3);  // 最多 3 条
  }

  private getCuisineName(cuisine: string): string {
    const map: Record<string, string> = {
      'sichuan': '川菜',
      'cantonese': '粤菜',
      'hunan': '湘菜',
      'jiangzhe': '江浙菜',
      'japanese': '日料',
      'korean': '韩餐'
    };
    return map[cuisine] || cuisine;
  }
}
```

---

### 3.6 营养数据库建设（MVP 方案）

#### 3.6.1 数据来源策略

**第一阶段（上线前）：人工标注 100 道热门菜品**
- 覆盖主流菜系（川菜、粤菜、湘菜、江浙菜、日料、韩餐）
- 覆盖不同餐型（早餐、午餐、晚餐）
- 每道菜标注：
  - 主要食材及份量
  - 烹饪方式
  - 营养估算（热量、蛋白质、碳水、脂肪、钠）

**标注工具：** 简易后台管理系统（可后续开发）

**标注依据：**
- 《中国食物成分表》标准版第 6 版
- 参考薄荷健康、Keep 等 APP 的公开数据
- 同类菜品取平均值

#### 3.6.2 营养估算公式（简化版）

```python
# 示例：宫保鸡丁（一份约 400g）
ingredients = [
    {'name': '鸡胸肉', 'amount_g': 150, 'nutrition_per_100g': {
        'calories': 133, 'protein': 24.6, 'carbs': 2.5, 'fat': 1.9
    }},
    {'name': '花生米', 'amount_g': 30, 'nutrition_per_100g': {
        'calories': 574, 'protein': 24.8, 'carbs': 16.2, 'fat': 49.2
    }},
    {'name': '青椒', 'amount_g': 50, 'nutrition_per_100g': {
        'calories': 23, 'protein': 1.4, 'carbs': 4.6, 'fat': 0.3
    }},
    {'name': '干辣椒', 'amount_g': 10, 'nutrition_per_100g': {
        'calories': 282, 'protein': 13.5, 'carbs': 53.7, 'fat': 6.8
    }}
]

# 计算食材总营养
total = {'calories': 0, 'protein': 0, 'carbs': 0, 'fat': 0}
for ing in ingredients:
    factor = ing['amount_g'] / 100
    nutr = ing['nutrition_per_100g']
    total['calories'] += nutr['calories'] * factor
    total['protein'] += nutr['protein'] * factor
    total['carbs'] += nutr['carbs'] * factor
    total['fat'] += nutr['fat'] * factor

# 添加烹饪油盐估算
total['calories'] += 45  #
