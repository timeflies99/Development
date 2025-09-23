`python-dotenv` 是一个用于从 `.env` 文件中读取环境变量并将其加载到 `os.environ` 中的 Python 库。它使得在不同环境（开发、测试、生产）中管理配置变得更加简单和安全，避免了将敏感信息（如 API 密钥、数据库密码）硬编码在代码中或直接暴露在系统环境变量里。

### 主要功能

1.  **加载 `.env` 文件**：读取项目根目录（或指定路径）下的 `.env` 文件。
2.  **设置环境变量**：将 `.env` 文件中定义的键值对设置到 `os.environ` 中，使得程序可以通过 `os.getenv()` 或 `os.environ[]` 来访问它们。
3.  **简化配置管理**：方便地在不同部署环境中切换配置。

### 安装

使用 pip 安装：

```bash
pip install python-dotenv
```

### 基本用法

1.  **创建 `.env` 文件**

    在你的 Python 项目根目录下创建一个名为 `.env` 的文件。文件内容格式为 `KEY=VALUE`，每行一个变量。支持注释（以 `#` 开头）和空行。

    ```bash
    # .env 文件示例
    DEBUG=True
    DATABASE_URL=postgresql://user:password@localhost:5432/myapp
    SECRET_KEY=your_very_secret_key_here
    API_TOKEN=abc123def456ghi789
    # 这是一个注释
    ```

2.  **在 Python 代码中加载 `.env` 文件**

    在你的 Python 脚本或应用的入口文件（如 `app.py`, `main.py` 或项目的初始化文件）中，尽早调用 `load_dotenv()` 函数来加载环境变量。

    ```python
    from dotenv import load_dotenv
    import os

    # 加载 .env 文件中的环境变量
    load_dotenv()  # 默认查找当前目录下的 .env 文件

    # 现在可以通过 os.getenv() 访问这些变量
    debug_mode = os.getenv('DEBUG')
    db_url = os.getenv('DATABASE_URL')
    secret_key = os.getenv('SECRET_KEY')
    api_token = os.getenv('API_TOKEN')

    print(f"Debug Mode: {debug_mode}")
    print(f"Database URL: {db_url}")
    print(f"Secret Key: {secret_key}")
    print(f"API Token: {api_token}")
    ```

### 高级用法

*   **指定 `.env` 文件路径**

    如果 `.env` 文件不在当前目录，或者你想加载不同的文件（如 `.env.development`, `.env.production`），可以传递 `dotenv_path` 参数。

    ```python
    from dotenv import load_dotenv

    # 加载特定路径的 .env 文件
    load_dotenv('/full/path/to/your/.env')
    # 或者相对于当前脚本的路径
    load_dotenv('../config/.env.local')
    ```

*   **覆盖现有环境变量**

    默认情况下，如果系统环境变量中已经存在同名变量，`load_dotenv()` **不会**覆盖它。如果你想让 `.env` 文件中的值覆盖已存在的环境变量，可以设置 `override=True`。

    ```python
    from dotenv import load_dotenv

    load_dotenv(override=True) # .env 文件中的值会覆盖已存在的环境变量
    ```

*   **处理变量插值**

    `.env` 文件支持简单的变量插值（引用其他变量）。

    ```bash
    # .env
    DOMAIN=example.com
    EMAIL=admin@${DOMAIN} # 或者 admin@$DOMAIN
    URL=https://${DOMAIN}/api
    ```

    ```python
    from dotenv import load_dotenv
    import os

    load_dotenv()
    print(os.getenv('EMAIL')) # 输出: admin@example.com
    print(os.getenv('URL'))   # 输出: https://example.com/api
    ```

*   **使用 `find_dotenv()`**

    `find_dotenv()` 函数会从当前工作目录开始，递归向上查找 `.env` 文件，直到找到为止。这对于在项目子目录中运行脚本时很有用。

    ```python
    from dotenv import load_dotenv, find_dotenv
    import os

    # 自动查找 .env 文件并加载
    load_dotenv(find_dotenv())

    # 或者先找到路径再加载
    dotenv_path = find_dotenv()
    load_dotenv(dotenv_path)
    ```

*   **从字符串或流中加载**

    除了从文件加载，也可以直接从字符串内容加载。

    ```python
    from dotenv import load_dotenv
    from io import StringIO

    dotenv_content = """
    KEY1=value1
    KEY2=value2
    """

    # 从 StringIO 对象加载
    stream = StringIO(dotenv_content)
    load_dotenv(stream=stream)

    # 或者更直接地（较新版本支持）
    # load_dotenv(stream=StringIO(dotenv_content))
    ```

### 最佳实践和注意事项

1.  **`.gitignore`**：务必将 `.env` 文件添加到你的 `.gitignore` 文件中，以防止敏感信息被提交到版本控制系统（如 Git）。
2.  **尽早加载**：在应用启动的最开始就调用 `load_dotenv()`，确保后续代码能正确读取到环境变量。
3.  **提供 `.env.example`**：创建一个 `.env.example` 文件，包含项目所需的所有环境变量名（但不包含真实值或使用占位符），并将其提交到版本库。这有助于其他开发者或部署人员了解需要配置哪些变量。
4.  **生产环境**：在生产环境中，通常不推荐使用 `.env` 文件，而是直接通过操作系统的环境变量、容器编排工具（如 Docker Compose, Kubernetes）或云平台的密钥管理服务来设置环境变量。`python-dotenv` 主要用于开发和测试环境。
5.  **类型转换**：`os.getenv()` 返回的是字符串。如果需要布尔值、整数等，需要手动转换，例如 `DEBUG = os.getenv('DEBUG', 'False').lower() == 'true'`。
6.  **错误处理**：考虑使用 `os.getenv('KEY', 'default_value')` 提供默认值，或者在变量缺失时抛出异常，以避免程序因缺少配置而出现难以调试的错误。

### 示例项目结构

```
my_project/
├── .env                # <-- 你的本地配置文件 (添加到 .gitignore)
├── .env.example        # <-- 配置模板 (提交到 Git)
├── .gitignore          # <-- 包含 ".env"
├── app.py              # <-- 你的主应用文件
└── requirements.txt    # <-- 包含 "python-dotenv"
```

`app.py`:

```python
#!/usr/bin/env python3
from dotenv import load_dotenv
import os

# 加载环境变量
load_dotenv()

# 获取配置
DATABASE_URL = os.getenv("DATABASE_URL")
if not DATABASE_URL:
    raise ValueError("No DATABASE_URL set for application")

SECRET_KEY = os.getenv("SECRET_KEY", "fallback-secret-key-for-dev")

DEBUG = os.getenv("DEBUG", "False").lower() in ("true", "1", "t")

print(f"App starting with DEBUG={DEBUG}")
# ... 你的应用逻辑
```

`.env.example`:

```bash
# Rename this file to .env and fill in your actual values
DEBUG=True
DATABASE_URL=postgresql://username:password@host:port/database_name
SECRET_KEY=change_me_to_a_real_secret
API_KEY=your_api_key_here
```

使用 `python-dotenv` 是 Python 项目中管理环境配置的一种非常流行和推荐的做法。