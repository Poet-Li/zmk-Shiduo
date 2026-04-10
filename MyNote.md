# 本地运行环境搭建
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip
pip install west
west init -l app/
west update
west zephyr-export
west packages pip --install
west sdk install //如果运行不了，就跑下面那条
//west sdk install --personal-access-token "GitHubToken" //此处双引号要留着


# 编译示例
.\.venv\Scripts\Activate.ps1
cd app
west build -p -b planck//zmk
west build -p -b nice_nano//zmk -- -DSHIELD=corne_left
west build -p -d build/glide_left -b xiao_ble//zmk -- -DSHIELD=glide_left
west build -p -d build/glide_right -b xiao_ble//zmk -- -DSHIELD=glide_right

# 注意
注意xiao_ble的引脚编号，和我在eda里随手0-13排的是不一样的