ARG CUDA_VERSION=12.8.1
FROM nvidia/cuda:${CUDA_VERSION}-cudnn-devel-ubuntu22.04

ARG BUILD_TYPE=all
ARG CMAKE_BUILD_PARALLEL_LEVEL=2 # (unused now; safe to remove if desired)
ENV DEBIAN_FRONTEND=noninteractive \
    CUDA_HOME=/usr/local/cuda
# Add GKE default lib and bin locations.
ENV PATH="${PATH}:/usr/local/nvidia/bin" \
    LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/usr/local/nvidia/lib:/usr/local/nvidia/lib64"

RUN apt update && apt install wget -y && apt install software-properties-common -y \
 && add-apt-repository ppa:deadsnakes/ppa -y \
 && apt install python3.12-full -y \
 && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1 \
 && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 2 \
 && update-alternatives --set python3 /usr/bin/python3.12 \
 && wget https://bootstrap.pypa.io/get-pip.py \
 && python3 get-pip.py

# Set timezone and install required runtime/build packages for single-node inference
RUN echo 'tzdata tzdata/Areas select America' | debconf-set-selections \
 && echo 'tzdata tzdata/Zones/America select Los_Angeles' | debconf-set-selections \
 && apt-get update && apt-get install -y --no-install-recommends \
    tzdata \
    software-properties-common unzip \
    ca-certificates curl wget \
    git git-lfs \
    python3 python3-pip python3-dev libpython3-dev python3-venv \
    build-essential cmake \
    libnuma1 \
    locales \
 && ln -sf /usr/bin/python3 /usr/bin/python \
 && rm -rf /var/lib/apt/lists/* \
 && apt-get clean

# (single-node inference) Skip GDRCopy/IBGDA setup

# Clone and install SGLang
WORKDIR /sgl-workspace
RUN python3 -m pip install --no-cache-dir --upgrade pip setuptools wheel html5lib six \
 && git clone --depth=1 https://github.com/tc-mb/sglang -b support-MiniCPM-V4.0 \
 && cd sglang \
 && case "$CUDA_VERSION" in \
      12.6.1) CUINDEX=126 ;; \
      12.8.1) CUINDEX=128 ;; \
      *) echo "Unsupported CUDA version: $CUDA_VERSION" && exit 1 ;; \
    esac \
 && python3 -m pip install --no-cache-dir -e "python[${BUILD_TYPE}]" --extra-index-url https://download.pytorch.org/whl/cu${CUINDEX} \
 && python3 -m pip install --no-cache-dir nvidia-nccl-cu12==2.27.6 --force-reinstall --no-deps \
 && if [ "$CUDA_VERSION" = "12.8.1" ]; then \
      python3 -m pip install --no-cache-dir https://github.com/sgl-project/whl/releases/download/v0.2.9/sgl_kernel-0.2.9+cu128-cp39-abi3-manylinux2014_x86_64.whl --force-reinstall --no-deps ; \
    fi

# (single-node inference) Skip NVSHMEM/DeepEP build & install

# Minimal extra Python tools for runtime (none needed beyond pip/wheel)

# Remove developer tooling and IB/RDMA stack for pure inference image

# Skip Nsight Systems CLI for inference-only image

# Set up locale
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# No extra dev/test Python packages for inference

# Drop dev UX tools and custom CMake install

 

# (removed oh-my-zsh and plugins)

# Configure Vim
COPY --chown=root:root <<-"EOF" /root/.vimrc
function! Yank(text) abort
  let escape = system('yank', a:text)
  if v:shell_error
    echoerr escape
  else
    call writefile([escape], '/dev/tty', 'b')
  endif
endfunction

noremap <silent> <Leader>y y:<C-U>call Yank(@0)<CR>

" automatically run yank(1) whenever yanking in Vim
function! CopyYank() abort
  call Yank(join(v:event.regcontents, "\n"))
endfunction

autocmd TextYankPost * call CopyYank()

" Basic settings
set number
syntax on
set mouse=a
filetype indent on

" Indentation
set autoindent nosmartindent
set smarttab
set expandtab
set shiftwidth=4
set softtabstop=4

" Visual guides
set colorcolumn=120
highlight ColorColumn ctermbg=5

" Status line
set laststatus=2
set statusline=%<%f\ %h%m%r%=%{\"[\".(&fenc==\"\"?&enc:&fenc).((exists(\"+bomb\")\ &&\ &bomb)?\",B\":\"\").\"]\ \"}%k\ %-14.(%l,%c%V%)\ %P

" Backspace behavior
set backspace=2

" Encoding
set encoding=utf-8
set fileencoding=utf-8
EOF

# Configure tmux
COPY --chown=root:root <<-"EOF" /root/.tmux.conf
# Pane border styling
set -g pane-border-style fg='#742727',bg=black
set -g pane-active-border-style fg=red,bg=black

# Status bar styling
set -g status-style bg='#0C8A92',fg=black

# Change prefix key to backtick
set-option -g prefix `
unbind C-b
bind-key ` send-prefix

# Split panes using - and = with current path
unbind '"'
bind - splitw -v -c '#{pane_current_path}'
unbind '%'
bind = splitw -h -c '#{pane_current_path}'

# Vi mode settings
bind-key -T copy-mode-vi Y send-keys -X copy-pipe 'yank > #{pane_tty}'
set-window-option -g mode-keys vi

# Other settings
set-option -g escape-time 0
set-option -g base-index 1
set-window-option -g mouse on
EOF

# Configure Git
RUN git config --global core.editor "vim" \
    && git config --global core.whitespace "fix,-indent-with-non-tab,trailing-space,cr-at-eol" \
    && git config --global core.pager "diff-so-fancy | less --tabs=4 -RFX" \
    && git config --global color.ui true \
    && git config --global color."diff-highlight".oldNormal "red bold" \
    && git config --global color."diff-highlight".oldHighlight "red bold 52" \
    && git config --global color."diff-highlight".newNormal "green bold" \
    && git config --global color."diff-highlight".newHighlight "green bold 22" \
    && git config --global color.diff.meta "11" \
    && git config --global color.diff.frag "magenta bold" \
    && git config --global color.diff.commit "yellow bold" \
    && git config --global color.diff.old "red bold" \
    && git config --global color.diff.new "green bold" \
    && git config --global color.diff.whitespace "red reverse" \
    && git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset - %s %Cgreen(%cr) %C(bold blue)<%an>%Creset%C(auto)%d%Creset' --abbrev-commit --" \
    && git config --global http.sslVerify false \
    && git config --global pull.rebase true

# Configure zsh
COPY --chown=root:root <<-"EOF" /root/.zshrc
export ZSH="/root/.oh-my-zsh"

# Theme
ZSH_THEME="robbyrussell"

# Plugins
plugins=(
    git
    z
    zsh-autosuggestions
    zsh-syntax-highlighting
)

source $ZSH/oh-my-zsh.sh

# Aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias vi='vim'

# Enhanced history
HISTSIZE=10000
SAVEHIST=10000
setopt HIST_IGNORE_ALL_DUPS
setopt HIST_FIND_NO_DUPS
setopt INC_APPEND_HISTORY
EOF

RUN set -euxo ; \
    curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/bin

# Set workspace directory
WORKDIR /sgl-workspace/sglang
