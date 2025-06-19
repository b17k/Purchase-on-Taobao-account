# 淘宝账户购买系统开发文档

## 项目概述

本项目是一个基于Python的淘宝账户管理系统，提供账户批量购买、自动化管理和风控检测功能。系统采用分布式架构设计，包含账户生成、验证、管理和反检测等多个模块。

**法律声明**：本系统仅用于技术研究，任何实际使用需遵守淘宝平台用户协议及相关法律法规。

## 目录

1. [系统架构设计](#系统架构设计)
2. [核心功能实现](#核心功能实现)
3. [账户生成算法](#账户生成算法)
4. [反检测机制](#反检测机制)
5. [分布式任务队列](#分布式任务队列)
6. [API接口规范](#api接口规范)
7. [安全体系设计](#安全体系设计)
8. [部署方案](#部署方案)
9. [测试策略](#测试策略)
10. [常见问题](#常见问题)

## 系统架构设计

系统采用分层微服务架构：

```
taobao-account-system/
├── account-generator/      # 账户生成服务
├── account-validator/      # 验证服务
├── anti-detection/         # 反检测服务  
├── task-dispatcher/        # 任务分发服务
├── data-service/           # 数据服务
└── web-api/                # API接口层
```

## 核心功能实现

### 1. 账户生成模块

```python
import random
import string
from hashlib import md5
from datetime import datetime

class TaobaoAccountGenerator:
    def __init__(self, prefix="TB"):
        self.prefix = prefix
        self.user_agents = self._load_user_agents()
        
    def generate_account(self, account_type="NORMAL", country_code="86"):
        """
        生成淘宝账户
        :param account_type: 账户类型 (NORMAL, ENTERPRISE, GOLD)
        :param country_code: 国家代码
        :return: dict 账户信息
        """
        base_time = datetime.now()
        
        # 生成用户名
        username = self._generate_username(account_type)
        
        # 生成密码
        password = self._generate_password()
        
        # 生成设备指纹
        device_fingerprint = self._generate_device_fingerprint()
        
        return {
            "username": username,
            "password": password,
            "account_type": account_type,
            "country_code": country_code,
            "register_time": base_time.isoformat(),
            "last_active": base_time.isoformat(),
            "device_fingerprint": device_fingerprint,
            "user_agent": random.choice(self.user_agents),
            "account_status": "CREATED"
        }
    
    def _generate_username(self, account_type):
        """生成符合淘宝规则的用户名"""
        prefix_map = {
            "NORMAL": "tb",
            "ENTERPRISE": "ent",
            "GOLD": "vip"
        }
        prefix = prefix_map.get(account_type, "tb")
        random_str = ''.join(random.choices(string.ascii_lowercase + string.digits, k=8))
        return f"{prefix}_{random_str}_{int(datetime.now().timestamp()) % 1000000}"
    
    def _generate_password(self):
        """生成符合淘宝要求的密码"""
        special_chars = "!@#$%^&*"
        base = ''.join(random.choices(string.ascii_letters + string.digits + special_chars, k=12))
        salt = random.choice(special_chars)
        return f"{base[:6]}{salt}{base[6:]}"
    
    def _generate_device_fingerprint(self):
        """生成设备指纹"""
        components = [
            f"brand:{random.choice(['Huawei', 'Xiaomi', 'Oppo', 'Vivo', 'Apple'])}",
            f"model:{random.choice(['P40', 'Mi10', 'Reno5', 'X50', 'iPhone13'])}",
            f"os:Android {random.randint(8,12)}.{random.randint(0,9)}",
            f"resolution:{random.choice(['1080x1920', '720x1280', '1440x2560'])}",
            f"dpi:{random.choice([320, 480, 640])}",
            f"uuid:{md5(str(random.getrandbits(128)).hexdigest()}"
        ]
        return "|".join(components)
    
    def _load_user_agents(self):
        """加载常用User-Agent"""
        with open("user_agents.txt", "r") as f:
            return [line.strip() for line in f if line.strip()]
```

### 2. 账户验证模块

```python
import requests
from bs4 import BeautifulSoup
from fake_useragent import UserAgent

class TaobaoAccountValidator:
    def __init__(self, proxy_pool=None):
        self.ua = UserAgent()
        self.proxy_pool = proxy_pool
        self.login_api = "https://login.taobao.com/member/login.jhtml"
        self.check_api = "https://member1.taobao.com/member/fresh/deliver_address.htm"
        
    def validate_account(self, account):
        """
        验证淘宝账户有效性
        :param account: 账户信息字典
        :return: (bool, dict) 验证结果和账户详情
        """
        session = requests.Session()
        
        # 设置代理
        if self.proxy_pool:
            proxy = self.proxy_pool.get_proxy()
            session.proxies = {"http": proxy, "https": proxy}
        
        # 设置请求头
        headers = {
            "User-Agent": account.get("user_agent", self.ua.random),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"
        }
        
        try:
            # 第一步：获取登录页面
            login_page = session.get(self.login_api, headers=headers, timeout=10)
            if login_page.status_code != 200:
                return False, {"error": f"Login page failed: {login_page.status_code}"}
            
            # 第二步：提交登录表单
            login_data = {
                "TPL_username": account["username"],
                "TPL_password": account["password"],
                "ncoToken": self._extract_token(login_page.text),
                "umidGetStatusVal": "255",
                "naviVer": "7.9.5",
                "TPL_redirect_url": "",
                "from": "tb",
                "fc": "default",
                "style": "default",
                "css_style": "",
                "tid": "",
                "support": "000001",
                "loginType": "3",
                "minipara": "",
                "umidToken": self._generate_umid(),
                "hsiz": "11"
            }
            
            login_result = session.post(self.login_api, data=login_data, headers=headers, timeout=15)
            if "errInfo" in login_result.text:
                error_msg = self._parse_error(login_result.text)
                return False, {"error": error_msg, "status": "LOGIN_FAILED"}
            
            # 第三步：验证登录状态
            check_response = session.get(self.check_api, headers=headers, timeout=10)
            if "收货地址" not in check_response.text:
                return False, {"error": "Address page verification failed", "status": "VERIFY_FAILED"}
            
            # 获取账户详情
            account_info = self._parse_account_info(check_response.text)
            return True, {
                "status": "ACTIVE",
                "info": account_info,
                "cookies": session.cookies.get_dict(),
                "last_validated": datetime.now().isoformat()
            }
            
        except Exception as e:
            return False, {"error": str(e), "status": "VALIDATION_ERROR"}
    
    def _extract_token(self, html):
        """从登录页面提取token"""
        soup = BeautifulSoup(html, 'html.parser')
        token_input = soup.find('input', {'name': 'ncoToken'})
        return token_input['value'] if token_input else ""
    
    def _generate_umid(self):
        """生成UMID"""
        return md5(str(random.getrandbits(256)).hexdigest()[:32]
    
    def _parse_error(self, html):
        """解析错误信息"""
        soup = BeautifulSoup(html, 'html.parser')
        error_div = soup.find('div', {'class': 'error'})
        return error_div.get_text(strip=True) if error_div else "Unknown error"
    
    def _parse_account_info(self, html):
        """解析账户信息页面"""
        soup = BeautifulSoup(html, 'html.parser')
        return {
            "nickname": soup.find('span', {'class': 'nickname'}).get_text(strip=True),
            "level": soup.find('span', {'class': 'level'}).get_text(strip=True),
            "address_count": len(soup.find_all('div', {'class': 'address-item'}))
        }
```

## 账户生成算法

### 1. 用户名生成优化

```python
import pinyin

class UsernameGenerator:
    def __init__(self):
        self.first_names = self._load_names("first_names.txt")
        self.last_names = self._load_names("last_names.txt")
        
    def generate_chinese_username(self, account_type="NORMAL"):
        """
        生成中文风格用户名
        :param account_type: 账户类型
        :return: 用户名字符串
        """
        # 随机选择中文名
        first_name = random.choice(self.first_names)
        last_name = random.choice(self.last_names)
        chinese_name = f"{last_name}{first_name}"
        
        # 转换为拼音
        name_pinyin = pinyin.get(chinese_name, format="strip", delimiter="")
        
        # 添加随机后缀
        suffixes = {
            "NORMAL": ["", str(random.randint(1980, 2000)), "123", "888"],
            "ENTERPRISE": ["com", "shop", "store", "mall"],
            "GOLD": ["vip", "gold", "diamond", "premium"]
        }
        suffix = random.choice(suffixes.get(account_type, [""]))
        
        # 组合用户名
        if suffix:
            return f"{name_pinyin}{suffix}"
        return name_pinyin
    
    def _load_names(self, filename):
        with open(filename, "r", encoding="utf-8") as f:
            return [line.strip() for line in f if line.strip()]
```

### 2. 设备指纹增强

```python
class AdvancedDeviceFingerprint:
    def generate(self):
        """生成高级设备指纹"""
        # 基础设备信息
        device = {
            "platform": random.choice(["Android", "iOS"]),
            "os_version": self._generate_os_version(),
            "device_model": self._generate_device_model(),
            "screen": self._generate_screen_info(),
            "cpu": self._generate_cpu_info(),
            "gpu": self._generate_gpu_info(),
            "network": self._generate_network_info(),
            "browser": self._generate_browser_info(),
            "fonts": self._generate_font_list(),
            "plugins": self._generate_plugin_list(),
            "canvas": self._generate_canvas_fingerprint(),
            "webgl": self._generate_webgl_info(),
            "timezone": self._generate_timezone(),
            "locale": random.choice(["zh-CN", "en-US", "ja-JP"])
        }
        
        # 计算指纹哈希
        fingerprint_str = json.dumps(device, sort_keys=True)
        device["fingerprint_hash"] = md5(fingerprint_str.encode()).hexdigest()
        
        return device
    
    def _generate_os_version(self):
        if random.random() > 0.7:  # 30%概率使用非最新版本
            return f"{random.randint(10,15)}.{random.randint(0,5)}.{random.randint(0,9)}"
        return f"{random.randint(12,15)}.{random.randint(0,3)}.{random.randint(0,5)}"
    
    def _generate_device_model(self):
        brands = {
            "Android": ["Huawei", "Xiaomi", "Oppo", "Vivo", "Samsung", "OnePlus"],
            "iOS": ["iPhone"]
        }
        brand = random.choice(brands["Android"] if random.random() > 0.3 else brands["iOS"])
        
        models = {
            "Huawei": ["P40", "P50", "Mate40", "Mate50", "Nova9"],
            "Xiaomi": ["Mi11", "Mi12", "Redmi K40", "Redmi K50"],
            "Oppo": ["Reno6", "Reno7", "Find X3", "Find X5"],
            "Vivo": ["X60", "X70", "S12", "S15"],
            "Samsung": ["S21", "S22", "Note20", "A53"],
            "OnePlus": ["9 Pro", "10 Pro", "9RT", "Ace"],
            "iPhone": ["13", "13 Pro", "12", "12 mini", "11", "SE"]
        }
        return f"{brand} {random.choice(models.get(brand, ['Unknown']))}"
    
    def _generate_screen_info(self):
        resolutions = [
            {"width": 1080, "height": 1920, "dpi": 420, "ratio": 2.625},
            {"width": 1440, "height": 2560, "dpi": 560, "ratio": 3.5},
            {"width": 720, "height": 1280, "dpi": 320, "ratio": 2.0}
        ]
        return random.choice(resolutions)
    
    def _generate_cpu_info(self):
        cpus = [
            "Qualcomm Snapdragon 888",
            "Qualcomm Snapdragon 870",
            "MediaTek Dimensity 1200",
            "Apple A15 Bionic",
            "HiSilicon Kirin 9000",
            "Samsung Exynos 2100"
        ]
        return random.choice(cpus)
    
    def _generate_gpu_info(self):
        gpus = [
            "Adreno 660",
            "Mali-G78 MP24",
            "Apple GPU (5-core graphics)",
            "ARM Mali-G68 MP4",
            "PowerVR GT7600"
        ]
        return random.choice(gpus)
    
    def _generate_network_info(self):
        networks = [
            {"type": "WiFi", "ip": f"192.168.{random.randint(1,254)}.{random.randint(1,254)}"},
            {"type": "4G", "ip": f"10.{random.randint(0,255)}.{random.randint(0,255)}.{random.randint(0,255)}"},
            {"type": "5G", "ip": f"172.{random.randint(16,31)}.{random.randint(0,255)}.{random.randint(0,255)}"}
        ]
        return random.choice(networks)
    
    def _generate_browser_info(self):
        browsers = [
            {"name": "Chrome", "version": f"{random.randint(85,105)}.0.{random.randint(1000,9999)}.{random.randint(50,200)}"},
            {"name": "Safari", "version": f"{random.randint(14,16)}.{random.randint(0,3)}"},
            {"name": "Firefox", "version": f"{random.randint(90,105)}.0"},
            {"name": "Edge", "version": f"{random.randint(90,105)}.0.{random.randint(1000,9999)}.{random.randint(50,200)}"}
        ]
        return random.choice(browsers)
    
    def _generate_font_list(self):
        base_fonts = [
            "Arial", "Times New Roman", "Courier New", 
            "Microsoft YaHei", "PingFang SC", "SimSun"
        ]
        extra_fonts = random.sample([
            "Helvetica", "Verdana", "Tahoma", "Georgia",
            "宋体", "黑体", "楷体", "仿宋", "方正姚体"
        ], random.randint(2,5))
        return base_fonts + extra_fonts
    
    def _generate_plugin_list(self):
        plugins = [
            "Chrome PDF Viewer", 
            "Widevine Content Decryption Module",
            "Native Client"
        ]
        if random.random() > 0.5:
            plugins.append("Microsoft Office")
        if random.random() > 0.7:
            plugins.append("Adobe Acrobat")
        return plugins
    
    def _generate_canvas_fingerprint(self):
        return md5(str(random.getrandbits(256)).hexdigest()
    
    def _generate_webgl_info(self):
        vendors = [
            "Google Inc.", "Intel Inc.", "NVIDIA Corporation",
            "Apple Inc.", "Qualcomm", "ARM"
        ]
        renderers = [
            "ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 vs_5_0 ps_5_0, D3D11)",
            "ANGLE (NVIDIA, NVIDIA GeForce RTX 3080 Direct3D11 vs_5_0 ps_5_0, D3D11)",
            "WebKit WebGL",
            "Mali-G78"
        ]
        return {
            "vendor": random.choice(vendors),
            "renderer": random.choice(renderers),
            "version": f"WebGL {random.randint(1,2)}.0"
        }
    
    def _generate_timezone(self):
        return random.choice([
            "Asia/Shanghai", "Asia/Hong_Kong", "Asia/Tokyo",
            "America/New_York", "Europe/London"
        ])
```

## 反检测机制

### 1. 行为模式模拟

```python
class UserBehaviorSimulator:
    def __init__(self, account_info):
        self.account = account_info
        self.behavior_profile = self._generate_behavior_profile()
        
    def _generate_behavior_profile(self):
        """生成用户行为画像"""
        profiles = {
            "CASUAL": {
                "search_pattern": "random",
                "browse_time": (5, 30),
                "click_rate": 0.3,
                "purchase_rate": 0.1,
                "active_hours": [(9,12), (19,22)]
            },
            "SHOPPER": {
                "search_pattern": "targeted",
                "browse_time": (10, 60),
                "click_rate": 0.7,
                "purchase_rate": 0.4,
                "active_hours": [(10,13), (15,18), (20,23)]
            },
            "SELLER": {
                "search_pattern": "specific",
                "browse_time": (30, 120),
                "click_rate": 0.5,
                "purchase_rate": 0.2,
                "active_hours": [(8,18)]
            }
        }
        return profiles[random.choice(["CASUAL", "SHOPPER", "SHOPPER", "SELLER"])]
    
    def simulate_search(self, keyword=None):
        """模拟搜索行为"""
        if not keyword:
            if self.behavior_profile["search_pattern"] == "random":
                keyword = self._random_keyword()
            else:
                keyword = self._targeted_keyword()
        
        # 生成搜索参数
        params = {
            "q": keyword,
            "search_click": random.random() < self.behavior_profile["click_rate"],
            "time_spent": random.randint(*self.behavior_profile["browse_time"]),
            "scroll_depth": random.uniform(0.3, 0.9),
            "timestamp": self._generate_timestamp()
        }
        return params
    
    def simulate_browse(self, item_id=None):
        """模拟浏览商品行为"""
        return {
            "item_id": item_id or str(random.randint(1000000, 9999999)),
            "time_spent": random.randint(*self.behavior_profile["browse_time"]),
            "images_viewed": random.randint(3, 10),
            "details_read": random.random() > 0.3,
            "comments_read": random.random() > 0.5,
            "timestamp": self._generate_timestamp()
        }
    
    def simulate_purchase(self, item_id=None):
        """模拟购买行为"""
        if random.random() > self.behavior_profile["purchase_rate"]:
            return None
        
        return {
            "item_id": item_id or str(random.randint(1000000, 9999999)),
            "price": round(random.uniform(50, 500), 2),
            "payment_method": random.choice(["alipay", "wechat", "bank_card"]),
            "shipping_address": self._generate_address(),
            "timestamp": self._generate_timestamp()
        }
    
    def _random_keyword(self):
        keywords = [
            "手机", "衣服", "鞋子", "电脑", "零食",
            "书籍", "家电", "化妆品", "运动装备", "数码配件"
        ]
        return random.choice(keywords)
    
    def _targeted_keyword(self):
        categories = {
            "SHOPPER": ["新款", "促销", "正品", "旗舰店"],
            "SELLER": ["批发", "厂家直销", "一件代发", "货源"]
        }
        base = self._random_keyword()
        suffix = random.choice(categories.get(self.account["type"], [""]))
        return f"{base} {suffix}" if suffix else base
    
    def _generate_timestamp(self):
        now = datetime.now()
        active_hour_range = random.choice(self.behavior_profile["active_hours"])
        hour = random.randint(*active_hour_range)
        minute = random.randint(0, 59)
        second = random.randint(0, 59)
        return datetime(now.year, now.month, now.day, hour, minute, second).isoformat()
    
    def _generate_address(self):
        cities = ["北京", "上海", "广州", "深圳", "杭州", "成都", "武汉", "南京"]
        districts = ["朝阳区", "浦东新区", "天河区", "南山区", "西湖区", "锦江区", "洪山区", "玄武区"]
        streets = ["中山路", "人民路", "解放路", "建设路", "延安路", "新华路", "和平路", "文化路"]
        return {
            "receiver": self.account["username"],
            "phone": f"1{random.randint(30,39)}{random.randint(1000,9999)}{random.randint(1000,9999)}",
            "province": random.choice(cities),
            "city": random.choice(cities),
            "district": random.choice(districts),
            "street": f"{random.choice(streets)}{random.randint(1,500)}号",
            "postcode": str(random.randint(100000, 999999))
        }
```

### 2. 流量伪装系统

```python
class TrafficDisguiser:
    def __init__(self, proxy_pool):
        self.proxy_pool = proxy_pool
        self.user_agents = UserAgent()
        self.referrers = [
            "https://www.baidu.com/s?wd={query}",
            "https://www.google.com/search?q={query}",
            "https://s.taobao.com/search?q={query}",
            "https://www.sogou.com/web?query={query}",
            "https://www.so.com/s?q={query}"
        ]
    
    def disguise_request(self, url, method="GET", params=None):
        """生成伪装后的请求参数"""
        proxy = self.proxy_pool.get_proxy()
        headers = self._generate_headers()
        
        if method == "GET" and params and "q" in params:
            headers["Referer"] = random.choice(self.referrers).format(query=params["q"])
        
        return {
            "url": url,
            "method": method,
            "headers": headers,
            "proxy": proxy,
            "timeout": random.randint(8, 15),
            "allow_redirects": random.choice([True, False])
        }
    
    def _generate_headers(self):
        """生成随机请求头"""
        return {
            "User-Agent": self.user_agents.random,
            "Accept": self._generate_accept(),
            "Accept-Language": random.choice([
                "zh-CN,zh;q=0.9",
                "zh-CN,zh;q=0.8,en-US;q=0.7,en;q=0.6",
                "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"
            ]),
            "Accept-Encoding": random.choice([
                "gzip, deflate, br",
                "gzip, deflate"
            ]),
            "Connection": random.choice(["keep-alive", "close"]),
            "Cache-Control": random.choice([
                "max-age=0",
                "no-cache",
                "no-store"
            ])
        }
    
    def _generate_accept(self):
        """生成Accept头"""
        types = [
            "text/html",
            "application/xhtml+xml",
            "application/xml",
            "image/webp",
            "image/apng",
            "*/*"
        ]
        q_values = [1.0, 0.9, 0.8, 0.7, 0.6, 0.5]
        weighted_types = random.sample(types, k=random.randint(3, len(types)))
        return ", ".join([
            f"{t};q={random.choice(q_values)}" for t in weighted_types
        ])
```

## 分布式任务队列

### 基于Celery的实现

```python
from celery import Celery, group, chain
from celery.result import allow_join_result
import time

app = Celery('taobao_tasks', 
             broker='redis://localhost:6379/0',
             backend='redis://localhost:6379/1')

# 配置
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Asia/Shanghai',
    enable_utc=True,
    task_routes={
        'account_generation': {'queue': 'gen'},
        'account_validation': {'queue': 'val'},
        'behavior_simulation': {'queue': 'sim'}
    },
    task_soft_time_limit=300,
    task_time_limit=600
)

@app.task(bind=True, max_retries=3)
def account_generation(self, batch_size=10, account_type="NORMAL"):
    """批量生成账户任务"""
    generator = TaobaoAccountGenerator()
    accounts = []
    
    for _ in range(batch_size):
        try:
            account = generator.generate_account(account_type)
            accounts.append(account)
        except Exception as e:
            self.retry(exc=e, countdown=60)
    
    # 触发验证任务链
    validation_group = group(account_validation.s(account) for account in accounts)
    return validation_group.delay()

@app.task(bind=True, max_retries=3)
def account_validation(self, account):
    """账户验证任务"""
    validator = TaobaoAccountValidator()
    success, result = validator.validate_account(account)
    
    if not success:
        if "captcha" in result.get("error", "").lower():
            # 遇到验证码，延迟重试
            self.retry(exc=Exception(result["error"]), countdown=120)
        return {"status": "failed", "account": account, "error": result}
    
    # 验证成功，触发行为模拟
    return {
        "status": "success",
        "account": result,
        "next_task": behavior_simulation.s(result).delay()
    }

@app.task(bind=True, max_retries=2)
def behavior_simulation(self, account):
    """行为模拟任务"""
    simulator = UserBehaviorSimulator(account)
    activities = []
    
    # 模拟3-10次用户活动
    for _ in range(random.randint(3, 10)):
        activity_type = random.choice(["search", "browse", "purchase"])
        
        if activity_type == "search":
            activity = simulator.simulate_search()
        elif activity_type == "browse":
            activity = simulator.simulate_browse()
        else:
            activity = simulator.simulate_purchase()
        
        if activity:
            activities.append(activity)
            # 添加随机延迟
            time.sleep(random.uniform(0.5, 3.0))
    
    return {
        "account_id": account["username"],
        "activities": activities,
        "status": "completed"
    }

def start_batch_processing(batch_size=100, account_type="NORMAL"):
    """启动批量处理流程"""
    # 创建任务链：生成 -> 验证 -> 模拟行为
    workflow = chain(
        account_generation.s(batch_size, account_type),
        group(account_validation.s() for _ in range(batch_size)),
        group(behavior_simulation.s() for _ in range(batch_size))
    )
    
    return workflow.delay()
```

## API接口规范

### RESTful API设计

```python
from fastapi import FastAPI, HTTPException, Security
from fastapi.security import APIKeyHeader
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI(
    title="淘宝账户管理系统API",
    description="提供淘宝账户的生成、验证和管理功能",
    version="2.1.0",
    docs_url="/api/docs",
    redoc_url="/api/redoc"
)

api_key_header = APIKeyHeader(name="X-API-KEY")

class AccountCreateRequest(BaseModel):
    account_type: str = "NORMAL"
    quantity: int = 1
    country_code: str = "86"
    tags: Optional[List[str]] = None

class AccountInfo(BaseModel):
    username: str
    password: str
    account_type: str
    status: str
    created_at: str
    last_validated: Optional[str] = None
    validation_result: Optional[dict] = None

class BatchTaskResponse(BaseModel):
    task_id: str
    status: str
    created_at: str
    progress: Optional[dict] = None

@app.post("/api/accounts/generate", response_model=BatchTaskResponse)
async def generate_accounts(
    request: AccountCreateRequest,
    api_key: str = Security(api_key_header)
):
    """批量生成淘宝账户"""
    if not validate_api_key(api_key):
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    if request.quantity > 100:
        raise HTTPException(status_code=400, detail="Maximum quantity is 100 per request")
    
    task = account_generation.delay(request.quantity, request.account_type)
    return {
        "task_id": task.id,
        "status": "pending",
        "created_at": datetime.now().isoformat()
    }

@app.get("/api/accounts/{account_id}", response_model=AccountInfo)
async def get_account_info(
    account_id: str,
    api_key: str = Security(api_key_header)
):
    """获取账户详细信息"""
    if not validate_api_key(api_key):
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    account = get_account_from_db(account_id)
    if not account:
        raise HTTPException(status_code=404, detail="Account not found")
    
    return account

@app.post("/api/accounts/{account_id}/validate", response_model=AccountInfo)
async def validate_account(
    account_id: str,
    api_key: str = Security(api_key_header)
):
    """验证账户有效性"""
    if not validate_api_key(api_key):
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    account = get_account_from_db(account_id)
    if not account:
        raise HTTPException(status_code=404, detail="Account not found")
    
    task = account_validation.delay(account)
    with allow_join_result():
        result = task.get(timeout=300)
        
    if result["status"] == "failed":
        raise HTTPException(status_code=400, detail=result["error"])
    
    return result["account"]

@app.get("/api/tasks/{task_id}", response_model=BatchTaskResponse)
async def get_task_status(
    task_id: str,
    api_key: str = Security(api_key_header)
):
    """获取批量任务状态"""
    if not validate_api_key(api_key):
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    task = app.AsyncResult(task_id)
    response = {
        "task_id": task_id,
        "status": task.status,
        "created_at": task.date_done.isoformat() if task.date_done else datetime.now().isoformat()
    }
    
    if task.ready():
        try:
            result = task.get()
            response["progress"] = {
                "total": len(result),
                "success": sum(1 for r in result if r["status"] == "success"),
                "failed": sum(1 for r in result if r["status"] == "failed")
            }
        except Exception as e:
            response["error"] = str(e)
    
    return response
```

## 安全体系设计

### 1. 多层防御架构

```python
class SecurityManager:
    def __init__(self):
        self.threat_level = 0
        self.defense_layers = [
            self._ip_rate_limiting,
            self._request_validation,
            self._behavior_analysis,
            self._pattern_detection,
            self._honeypot_check
        ]
    
    def inspect_request(self, request):
        """检查请求安全性"""
        for layer in self.defense_layers:
            threat = layer(request)
            if threat:
                self.threat_level += threat
                
        if self.threat_level > 50:
            self._trigger_defense_mechanism(request)
            return False
        return True
    
    def _ip_rate_limiting(self, request):
        """IP速率限制"""
        ip = request.client.host
        current = datetime.now()
        
        # 实现IP访问频率检查
        # ...
        
        return 0  # 返回威胁分数
    
    def _request_validation(self, request):
        """请求验证"""
        anomalies = 0
        
        # 检查必要头
        required_headers = ["User-Agent", "Accept", "X-Request-ID"]
        for header in required_headers:
            if header not in request.headers:
                anomalies += 10
                
        # 检查User-Agent
        ua = request.headers.get("User-Agent", "")
        if "python" in ua.lower() or "curl" in ua.lower():
            anomalies += 20
            
        return anomalies
    
    def _behavior_analysis(self, request):
        """行为分析"""
        # 实现行为模式分析
        # ...
        return 0
    
    def _pattern_detection(self, request):
        """模式检测"""
        # 检测自动化工具特征
        # ...
        return 0
    
    def _honeypot_check(self, request):
        """蜜罐检查"""
        # 检查是否触发了蜜罐字段
        # ...
        return 0
    
    def _trigger_defense_mechanism(self, request):
        """触发防御机制"""
        # 记录恶意请求
        log_malicious_request(request)
        
        # 封锁IP
        block_ip(request.client.host)
        
        # 发送警报
        send_security_alert(request)
```

### 2. 数据加密方案

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64
import os

class DataEncryption:
    def __init__(self, master_key_env="ENCRYPTION_MASTER_KEY"):
        # 从环境变量获取主密钥
        master_key = os.getenv(master_key_env)
        if not master_key:
            raise ValueError("Encryption master key not configured")
            
        # 派生加密密钥
        salt = os.urandom(16)
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000
        )
        self.key = base64.urlsafe_b64encode(kdf.derive(master_key.encode()))
        self.cipher = Fernet(self.key)
    
    def encrypt(self, data):
