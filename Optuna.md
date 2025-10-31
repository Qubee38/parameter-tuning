## 3. Optuna：高度なパラメータ探索

### 3.1 特徴

- **開発元**: Preferred Networks（Python実装）
- **制約**: 境界制約、一般制約はペナルティ法で対応
- **探索方法**: ベイズ最適化（Tree-structured Parzen Estimator）
- **長所**: 大域解を見つけやすい、並列実行、可視化機能
- **短所**: Pythonが必要、実行時間が長い

### 3.2 Fortranとの連携方法

Optunaは**Pythonで最適化ロジックを記述**し、**Fortranプログラムを外部から呼び出す**方式です。

```
┌─────────────────────┐
│  Python (Optuna)    │
│  - パラメータ管理    │
│  - 最適化ロジック    │
│  - 結果の可視化      │
└──────┬──────────────┘
       │ パラメータ送信
       ▼
┌─────────────────────┐
│  Fortranプログラム   │
│  - 物理モデル計算    │
│  - RSS計算          │
└──────┬──────────────┘
       │ 結果返却
       ▼
     繰り返し
```

### 3.3 実装例

コード例を示します。

#### Fortranプログラム（physics_model.f90）

```fortran
program physics_model
  implicit none
  real(8) :: params(5)
  real(8) :: rss
  integer :: i
  
  ! Pythonから送られたパラメータを読み込み
  open(10, file='params.txt', status='old')
  do i = 1, 5
    read(10, *) params(i)
  end do
  close(10)
  
  ! 物理モデルの計算とRSSの算出
  call compute_rss(params, rss)
  
  ! 結果をPythonに返す
  open(20, file='result.txt', status='replace')
  write(20, *) rss
  close(20)
  
contains

  subroutine compute_rss(params, rss)
    real(8) :: params(5), rss
    integer, parameter :: n_obs = 100
    real(8) :: obs_data(n_obs), obs_time(n_obs)
    real(8) :: model_output(n_obs)
    
    ! 観測データの読み込み
    call load_observation_data(obs_data, obs_time, n_obs)
    
    ! 物理モデルの計算
    call physics_simulation(params, obs_time, model_output, n_obs)
    
    ! 残差二乗和
    rss = sum((obs_data - model_output)**2)
    
  end subroutine compute_rss
  
  ! ここに物理シミュレーションのコードを記述
  subroutine physics_simulation(params, time, output, n)
    integer :: n
    real(8) :: params(5), time(n), output(n)
    ! 物理モデルの実装
  end subroutine physics_simulation
  
  subroutine load_observation_data(data, time, n)
    integer :: n
    real(8) :: data(n), time(n)
    ! 観測データの読み込み
  end subroutine load_observation_data
  
end program physics_model
```

#### Pythonスクリプト（optimize.py）

```python
import optuna
import subprocess

def objective(trial):
    """
    Optunaが呼び出す目的関数
    """
    # パラメータの提案
    params = {
        'param1': trial.suggest_float('param1', 0.01, 5.0, log=True),
        'param2': trial.suggest_float('param2', 10.0, 1000.0, log=True),
        'param3': trial.suggest_float('param3', 0.0, 1.0),
        'param4': trial.suggest_float('param4', 0.0, 40.0),
        'param5': trial.suggest_float('param5', 0.001, 0.5, log=True)
    }
    
    # パラメータをファイルに書き出し
    with open('params.txt', 'w') as f:
        for key in ['param1', 'param2', 'param3', 'param4', 'param5']:
            f.write(f'{params[key]}\n')
    
    # Fortranプログラムを実行
    result = subprocess.run(['./physics_model'], 
                          capture_output=True, 
                          text=True)
    
    if result.returncode != 0:
        print(f"Error: {result.stderr}")
        return float('inf')
    
    # 結果（RSS）を読み取り
    with open('result.txt', 'r') as f:
        rss = float(f.read().strip())
    
    return rss

# 最適化の実行
print("="*50)
print("Optunaによる最適化を開始します")
print("="*50)

# Studyの作成
study = optuna.create_study(
    direction='minimize',
    sampler=optuna.samplers.TPESampler(seed=42),
    study_name='physics_simulation'
)

# 最適化実行（並列処理可能）
study.optimize(objective, n_trials=200, n_jobs=4)

# 結果の表示
print("\n" + "="*50)
print("最適化完了")
print("="*50)
print(f"最小RSS: {study.best_value:.6f}")
print("最適パラメータ:")
for key, value in study.best_params.items():
    print(f"  {key}: {value:.6f}")

# 結果の保存
import pandas as pd
df = study.trials_dataframe()
df.to_csv('optimization_results.csv', index=False)

# 可視化
import optuna.visualization as vis
vis.plot_optimization_history(study).write_html('history.html')
vis.plot_param_importances(study).write_html('importance.html')

print("\n結果を保存しました:")
print("  - optimization_results.csv: 全試行の詳細")
print("  - history.html: 最適化の履歴")
print("  - importance.html: パラメータの重要度")
```

### 3.4 実行手順

```bash
# 1. Fortranプログラムのコンパイル
gfortran -O3 -o physics_model physics_model.f90

# 2. Optunaのインストール（初回のみ）
pip install optuna pandas plotly

# 3. 最適化の実行
python optimize.py
```

### 3.5 マルチスタート（複数回実行）

大域解をより確実に見つけるため、異なる条件で複数回実行：

```python
import optuna

def objective(trial):
    # ... （前述と同じ）
    return rss

# マルチスタート: 5回実行
n_runs = 5
all_results = []

for run_id in range(n_runs):
    print(f"\n=== 実行 {run_id + 1}/{n_runs} ===")
    
    # 異なるシードで新しいstudyを作成
    study = optuna.create_study(
        direction='minimize',
        sampler=optuna.samplers.TPESampler(seed=run_id)
    )
    
    # 最適化実行
    study.optimize(objective, n_trials=100, show_progress_bar=True)
    
    # 結果を保存
    all_results.append({
        'run_id': run_id,
        'best_value': study.best_value,
        'best_params': study.best_params
    })
    
    print(f"RSS: {study.best_value:.4f}")

# 最良の結果を選択
best_run = min(all_results, key=lambda x: x['best_value'])

print("\n" + "="*50)
print("最終結果（全実行中の最良）")
print("="*50)
print(f"実行ID: {best_run['run_id']}")
print(f"最小RSS: {best_run['best_value']:.4f}")
print(f"最適パラメータ: {best_run['best_params']}")
```