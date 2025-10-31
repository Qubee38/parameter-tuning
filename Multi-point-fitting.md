# 複数地点データの統合

- [複数地点データの統合](#複数地点データの統合)
  - [実装方法](#実装方法)
    - [方法1：単純合計](#方法1単純合計)
    - [方法2：重み付き合計](#方法2重み付き合計)
    - [方法3：正規化RSS](#方法3正規化rss)
    - [方法4：地点別パラメータと共通パラメータ](#方法4地点別パラメータと共通パラメータ)

## 実装方法

### 方法1：単純合計

全地点のRSSを単純に合計する方法（最もシンプル）：

```fortran
subroutine multi_site_objective(val, n, x, gradient, need_gradient, f_data)
  implicit none
  integer :: n, need_gradient
  double precision :: val, x(n), gradient(n)
  integer*8 :: f_data
  
  integer, parameter :: n_sites = 3  ! 地点数
  double precision :: rss_site
  integer :: i
  
  val = 0.0d0
  
  ! 各地点でRSSを計算して合計
  do i = 1, n_sites
    call compute_site_rss(i, x, rss_site)
    val = val + rss_site
  end do
  
end subroutine multi_site_objective

! 各地点のRSS計算
subroutine compute_site_rss(site_id, params, rss)
  implicit none
  integer :: site_id
  double precision :: params(5), rss
  
  integer :: n_obs
  double precision, allocatable :: obs_data(:), obs_time(:)
  double precision, allocatable :: model_output(:)
  character(len=100) :: filename
  
  ! 地点ごとの観測データを読み込み
  write(filename, '(A,I0,A)') 'data/site_', site_id, '.txt'
  call load_site_data(filename, obs_data, obs_time, n_obs)
  
  allocate(model_output(n_obs))
  
  ! モデル計算（地点固有の環境条件を考慮）
  call physics_model_with_site_conditions(site_id, params, &
                                         obs_time, model_output, n_obs)
  
  ! RSS計算
  rss = sum((obs_data - model_output)**2)
  
  deallocate(obs_data, obs_time, model_output)
  
end subroutine compute_site_rss
```

### 方法2：重み付き合計

データ数や観測精度が地点間で異なる場合に有効：

```fortran
subroutine weighted_multi_site_objective(val, n, x, gradient, &
                                        need_gradient, f_data)
  implicit none
  integer :: n, need_gradient
  double precision :: val, x(n), gradient(n)
  integer*8 :: f_data
  
  integer, parameter :: n_sites = 3
  double precision :: rss_site
  double precision :: weights(n_sites)
  integer :: i
  
  ! 重みの設定
  ! 方法1: データ数に反比例（データ数が多い地点の影響を抑える）
  weights(1) = 1.0d0 / 100.0d0  ! 地点A: 100データ点
  weights(2) = 1.0d0 / 50.0d0   ! 地点B: 50データ点
  weights(3) = 1.0d0 / 150.0d0  ! 地点C: 150データ点
  
  ! または、方法2: 観測精度に基づく
  ! weights = [1.0, 0.8, 0.5]  ! 地点Cは精度が低い
  
  val = 0.0d0
  
  do i = 1, n_sites
    call compute_site_rss(i, x, rss_site)
    
    ! 重み付きRSSの合計
    val = val + weights(i) * rss_site
  end do
  
end subroutine weighted_multi_site_objective
```

### 方法3：正規化RSS

地点間でデータのスケールが大きく異なる場合：

```fortran
subroutine normalized_multi_site_objective(val, n, x, gradient, &
                                          need_gradient, f_data)
  implicit none
  integer :: n, need_gradient
  double precision :: val, x(n), gradient(n)
  integer*8 :: f_data
  
  integer, parameter :: n_sites = 3
  double precision :: rss_site, nrss_site
  double precision :: variance_obs
  integer :: i
  
  val = 0.0d0
  
  do i = 1, n_sites
    ! 通常のRSS計算
    call compute_site_rss(i, x, rss_site)
    
    ! 観測データの分散を取得
    call get_site_variance(i, variance_obs)
    
    ! 正規化RSS = RSS / 分散
    nrss_site = rss_site / variance_obs
    
    val = val + nrss_site
  end do
  
end subroutine normalized_multi_site_objective
```

### 方法4：地点別パラメータと共通パラメータ

一部のパラメータを全地点で共通、一部を地点固有とする方法：

```fortran
! パラメータ構成:
! - 共通パラメータ（全地点共通）: 2個
! - 地点固有パラメータ: 各地点1個
! 合計: 2 + 3×1 = 5パラメータ

subroutine mixed_parameter_objective(val, n, x, gradient, &
                                    need_gradient, f_data)
  implicit none
  integer :: n, need_gradient
  double precision :: val, x(n), gradient(n)
  integer*8 :: f_data
  
  integer, parameter :: n_sites = 3
  double precision :: common_params(2)
  double precision :: site_params(5)
  double precision :: rss_site
  integer :: i
  
  ! 共通パラメータを抽出
  common_params(1) = x(1)  ! パラメータ1（全地点共通）
  common_params(2) = x(2)  ! パラメータ2（全地点共通）
  
  val = 0.0d0
  
  do i = 1, n_sites
    ! 各地点のパラメータセットを構成
    site_params(1) = common_params(1)  ! 共通
    site_params(2) = common_params(2)  ! 共通
    site_params(3) = 0.1d0             ! 固定値
    site_params(4) = x(2 + i)          ! 地点固有
    site_params(5) = 0.05d0            ! 固定値
    
    ! 地点のRSS計算
    call compute_site_rss(i, site_params, rss_site)
    val = val + rss_site
  end do
  
end subroutine mixed_parameter_objective
```