## 1. MINPACK：最小二乗法による最適化

### 1.1 特徴

- **開発元**: アルゴンヌ国立研究所
- **制約**: 境界制約、一般制約に対応
- **アルゴリズム**: Levenberg-Marquardt法
- **長所**: 高速、安定、実績豊富
- **短所**: 境界制約が使えない、局所解に収束する可能性

### 1.2 基本的な使い方

コード例を示します。

```fortran
program minpack_example
  implicit none
  
  ! パラメータ数と観測データ数
  integer, parameter :: n_params = 5      ! チューニングするパラメータ数
  integer, parameter :: n_obs = 100       ! 観測データ点数
  
  ! 変数宣言
  real(8) :: params(n_params)             ! パラメータベクトル
  real(8) :: fvec(n_obs)                  ! 残差ベクトル
  integer :: info
  
  ! MINPACK用の作業配列
  integer, parameter :: lwa = n_obs*n_params + 5*n_params + n_obs
  real(8) :: wa(lwa)
  integer :: iwa(n_params)
  real(8) :: tol
  
  ! 初期パラメータ値の設定（重要！）
  params(1) = 0.5     ! 例：光合成の最大速度
  params(2) = 100.0   ! 例：光飽和定数
  params(3) = 0.1     ! 例：呼吸速度
  params(4) = 20.0    ! 例：最適温度
  params(5) = 0.05    ! 例：温度依存性パラメータ
  
  ! 許容誤差
  tol = 1.0d-8
  
  ! MINPACK LMDIF1を呼び出し
  call lmdif1(fcn, n_obs, n_params, params, fvec, tol, info, &
              iwa, wa, lwa)
  
  ! 結果の出力
  if (info >= 1 .and. info <= 3) then
    print *, '最適化成功'
    print *, '最適パラメータ:'
    print *, params
    print *, '残差二乗和:', sum(fvec**2)
  else
    print *, '最適化失敗, info =', info
  end if
  
contains

  ! 目的関数：観測値とモデル値の残差を計算
  subroutine fcn(m, n, x, fvec, iflag)
    implicit none
    integer :: m, n, iflag
    real(8) :: x(n), fvec(m)
    real(8) :: obs_data(m), obs_time(m)
    real(8) :: model_output(m)
    
    ! 観測データの読み込み
    call load_observation_data(obs_data, obs_time, m)
    
    ! 物理モデルの計算
    call physics_model(x, obs_time, model_output, m)
    
    ! 残差の計算
    fvec = obs_data - model_output
    
  end subroutine fcn
  
  subroutine physics_model(params, time, output, n)
    integer :: n
    real(8) :: params(5), time(n), output(n)
    ! ここに物理モデルの計算を記述
    ! output = f(time, params)
  end subroutine physics_model
  
  subroutine load_observation_data(data, time, n)
    integer :: n
    real(8) :: data(n), time(n)
    ! ここに観測データの読み込みを記述
  end subroutine load_observation_data
  
end program minpack_example
```

### 1.3 コンパイルと実行

```bash
# MINPACKのダウンロード
wget http://www.netlib.org/minpack/minpack.tar.gz
tar -xzf minpack.tar.gz
cd minpack

# ライブラリのコンパイル
gfortran -c *.f
ar rcs libminpack.a *.o

# プログラムのコンパイルとリンク
gfortran -o optimize your_program.f90 -L. -lminpack

# 実行
./optimize
```
