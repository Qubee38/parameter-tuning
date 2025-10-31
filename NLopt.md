## 2. NLopt：制約付き最適化

### 2.1 特徴

- **開発元**: MIT（C言語実装、Fortranバインディング提供）
- **制約**: 境界制約、一般制約に対応
- **アルゴリズム**: 30種類以上から選択可能
- **長所**: 制約を自然に扱える、多様なアルゴリズム
- **短所**: インストールがやや複雑

### 2.2 推奨アルゴリズム

物理シミュレーション向けの推奨アルゴリズム：

| アルゴリズム | コード | 特徴 | 適用場面 |
|------------|--------|------|---------|
| **BOBYQA** | NLOPT_LN_BOBYQA | 境界制約、勾配不要 | **最も推奨** |
| COBYLA | NLOPT_LN_COBYLA | 一般制約、勾配不要 | 複雑な制約がある場合 |
| NEWUOA | NLOPT_LN_NEWUOA | 制約なし、勾配不要 | 制約不要な場合 |
| SLSQP | NLOPT_LD_SLSQP | 勾配ベース、高速 | 導関数が計算できる場合 |

### 2.3 基本的な使い方

コード例を示します。

```fortran
program nlopt_example
  implicit none
  include 'nlopt.f'
  
  ! 変数宣言
  integer*8 :: opt
  integer :: ier, n_params
  double precision :: params(5)
  double precision :: lower_bounds(5), upper_bounds(5)
  double precision :: minf
  
  n_params = 5
  
  ! 初期パラメータ
  params(1) = 0.5d0    ! 初期推定値
  params(2) = 100.0d0
  params(3) = 0.1d0
  params(4) = 20.0d0
  params(5) = 0.05d0
  
  ! 境界制約の設定（重要！）
  ! パラメータの物理的に妥当な範囲を設定
  lower_bounds(1) = 0.01d0   ! 光合成速度は正の値
  upper_bounds(1) = 2.0d0
  
  lower_bounds(2) = 10.0d0   ! 光飽和定数は正の値
  upper_bounds(2) = 1000.0d0
  
  lower_bounds(3) = 0.0d0    ! 呼吸速度は非負
  upper_bounds(3) = 1.0d0
  
  lower_bounds(4) = 0.0d0    ! 温度（℃）
  upper_bounds(4) = 40.0d0
  
  lower_bounds(5) = 0.001d0  ! 温度依存性は正の値
  upper_bounds(5) = 0.5d0
  
  print *, '========================================'
  print *, 'NLoptによる最適化'
  print *, '========================================'
  
  ! オプティマイザの作成（BOBYQA推奨）
  call nlo_create(opt, NLOPT_LN_BOBYQA, n_params)
  
  ! 境界制約の設定
  call nlo_set_lower_bounds(ier, opt, lower_bounds)
  call nlo_set_upper_bounds(ier, opt, upper_bounds)
  
  ! 目的関数の設定
  call nlo_set_min_objective(ier, opt, objective_function, 0)
  
  ! 収束基準の設定
  call nlo_set_xtol_rel(ier, opt, 1.0d-8)    ! 相対変化量
  call nlo_set_maxeval(ier, opt, 10000)      ! 最大評価回数
  
  ! 最適化実行
  print *, '最適化を開始します...'
  call nlo_optimize(ier, opt, params, minf)
  
  ! 結果の出力
  print *, ''
  if (ier .lt. 0) then
    print *, '最適化失敗, error code:', ier
  else
    print *, '最適化成功'
    print *, '最適パラメータ:'
    print *, '  パラメータ1:', params(1)
    print *, '  パラメータ2:', params(2)
    print *, '  パラメータ3:', params(3)
    print *, '  パラメータ4:', params(4)
    print *, '  パラメータ5:', params(5)
    print *, '最小RSS:', minf
  end if
  
  ! クリーンアップ
  call nlo_destroy(opt)
  
end program nlopt_example

! 目的関数
subroutine objective_function(val, n, x, gradient, need_gradient, f_data)
  implicit none
  integer :: n, need_gradient
  double precision :: val, x(n), gradient(n)
  integer*8 :: f_data
  
  integer, parameter :: n_obs = 100
  double precision :: obs_data(n_obs), obs_time(n_obs)
  double precision :: model_output(n_obs)
  
  ! 観測データの読み込み
  call load_observation_data(obs_data, obs_time, n_obs)
  
  ! 物理モデルの計算
  call physics_model(x, obs_time, model_output, n_obs)
  
  ! 残差二乗和の計算
  val = sum((obs_data - model_output)**2)
  
end subroutine objective_function
```

### 2.4 インストール

```bash
# Ubuntu/Debianの場合
sudo apt-get install libnlopt-dev

# ソースからビルド
git clone https://github.com/stevengj/nlopt.git
cd nlopt
mkdir build && cd build
cmake ..
make
sudo make install
```

### 2.5 コンパイル

```bash
# 基本的なコンパイル
gfortran -o optimize your_program.f90 -lnlopt -lm

# 最適化オプション付き
gfortran -O3 -o optimize your_program.f90 -lnlopt -lm

# インクルードパスの指定が必要な場合
gfortran -I/usr/local/include -L/usr/local/lib \
  -o optimize your_program.f90 -lnlopt -lm
```