# 1) 安装 rustup（若已装可跳过）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

# 2) 让当前 shell 生效
source /root/.cargo/env

# 3) 更新到最新稳定版（会带新 cargo）
rustup update stable
rustup default stable

# 4) 验证
cargo -V
rustc -V

# 5) 再装
cargo install ripgrep
