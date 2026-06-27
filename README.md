# F1 进站预测 - Stacking 融合竞赛方案
摘要：本项目为 F1 进站预测竞赛解决方案，基于圈级赛车数据构建二分类模型，预测下一圈赛车是否进站，核心评价指标为 ROC-AUC。项目采用官方 2021-2023 年 37 场大奖赛数据集，训练集约 15.2 万条样本，进站样本占比约 10%，存在显著类别不平衡。技术上围绕轮胎磨损、比赛进度搭建业务特征体系，结合分箱、目标编码与伪标签增强；选用 LightGBM、XGBoost、CatBoost 三款异构树模型，经 5 折交叉验证与多种子平均后，通过逻辑回归完成 Stacking 两层融合。最终线上 ROC-AUC 达 0.9521，可辅助车队优化进站策略，也为表格数据竞赛提供了可复用的实践思路。

（一）任务定义  
任务类型：不平衡二分类任务  
预测目标：给定当前圈的赛车数据，预测下一圈是否进站（1 = 进站，0 = 不进站）  
评价指标：ROC-AUC  
业务价值：进站策略是 F1 赛事胜负的核心影响因素，精准的进站时机预测可辅助车队优化轮胎策略、安全车应对与战术布局。  

（二）数据集说明  
官方提供 2021-2023 年 37 场 F1 大奖赛的圈级数据：  
训练集：152,347 条样本  
测试集：51,234 条样本  
原始特征：24 个，覆盖车手、轮胎、圈速、位置、比赛进度五大维度  
类别分布：进站样本占比 9.87%，存在显著类别不平衡  

（三）技术方案  
1. 数据预处理  
缺失值填充：数值特征填 0，类别特征填特殊标记 NA  
异常值处理：对核心数值特征采用 3σ 原则截断，降低极端值干扰  
特征编码：低基数特征标签编码，高基数特征结合频率编码与交叉验证目标编码  
2. 特征工程  
围绕 F1 业务规律构建多维度特征，最终特征规模 150+：  
基础衍生特征：比赛进度、剩余圈数、轮胎磨损率、相对圈速、位置变化率等  
业务分箱特征：基于赛事规则对轮胎寿命、比赛阶段、排名进行分箱，适配树模型切分  
数字位特征：提取核心数值的整数位、小数位，捕捉隐藏的周期性规律  
聚合统计特征：按赛道 - 轮胎、车手 - 轮胎等维度构建聚合统计特征  
目标编码特征：在 5 折交叉验证内完成带平滑的目标编码，严格避免数据泄露  
3. 数据增强  
伪标签技术：使用高置信度测试集预测结果扩充训练集，提升模型对边缘场景的泛化能力  
4. 模型选型  
对比多类模型后，选择三款异构树模型作为基础模型，充分利用算法间互补性：  
LightGBM：擅长高维稀疏特征与编码特征，训练效率高  
XGBoost：对数值特征与异常值鲁棒性强  
CatBoost：原生支持高基数类别特征，无需复杂预处理  
5. 模型融合  
采用两层 Stacking 融合策略：  
第一层：LightGBM / XGBoost / CatBoost 三个异构模型，5 折分层交叉验证生成 OOF 预测  
第二层：逻辑回归作为元模型，结构简单稳定，不易过拟合  
多随机种子平均：5 组随机种子训练结果取平均，进一步降低随机性、提升稳定性  

（四）项目结构  
f1-pit-prediction/  
├── data/                   # 数据集目录  
│   ├── train.csv  
│   ├── test.csv  
│   └── sample_submission.csv  
├── main.py                 # 主训练与预测脚本  
├── generate_plots.py       # 可视化图表生成脚本  
├── output/                 # 结果输出目录  
│   └── f1_pit_final.csv    # 最终提交文件  
└── README.md  

（五）环境依赖  
·Python 3.8+  
·pandas >= 1.3.0  
·numpy >= 1.21.0  
·scikit-learn >= 1.0.0  
·lightgbm >= 3.3.0  
·xgboost >= 1.5.0  
·catboost >= 1.0.0  
一键安装依赖：  
pip install pandas numpy scikit-learn lightgbm xgboost catboost  

（六）运行方式  
1.将官方数据集放入 data/ 目录  
2.修改 main.py 中的 DATA_PATH 与 SAVE_PATH 为本地对应路径  
3.运行主脚本：  
python main.py  
4.运行完成后，可提交的预测结果文件将生成在输出目录中  

（七）实验结果  
| 阶段 | 方案 | 线上 AUC | 提升幅度 |
| ---- | ---- | -------- | -------- |
| 1 | 原始特征 + LightGBM | 0.9436 | - |
| 2 | 基础衍生特征 | 0.9468 | +0.0032 |
| 3 | 完整特征工程 | 0.9492 | +0.0024 |
| 4 | 伪标签数据增强 | 0.9499 | +0.0007 |
| 5 | 三模型加权融合 | 0.9501 | +0.0002 |
| 6 | 三模型 Stacking 融合 | 0.9521 | +0.0007 |  
关键结论  
·特征工程是表格数据竞赛的核心增益点，人工业务特征 + 统计聚合特征组合效果最优  
·异构模型 Stacking 融合可有效挖掘模型间互补性，稳定提升最终得分  
·超参数调优需关注本地与线上一致性，基线有界精细调优效果优于无约束全局搜索  

（八）后续改进方向  
1.在网上搜集安全车、天气、赛道特性等外部维度数据  
2.针对安全车、雨战等罕见场景做专项数据增强与重采样  
3.探索 TabNet、FT-Transformer 等表格深度学习方案  
4.尝试多层 Stacking 与多粒度融合策略  

说明  
本项目为竞赛练习方案，代码与结果仅作技术交流使用    

## 附录：完整核心源码
```python
import pandas as pd
import numpy as np
import os
import warnings
from lightgbm import LGBMClassifier, early_stopping
from xgboost import XGBClassifier
from catboost import CatBoostClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold
from sklearn.preprocessing import LabelEncoder
from sklearn.metrics import roc_auc_score

warnings.filterwarnings('ignore')

# ===================== 基础配置 =====================
DATA_PATH = r"kaggle data"
SAVE_PATH = r"kaggle_final_version"
os.makedirs(SAVE_PATH, exist_ok=True)

SEEDS = [111, 222, 333, 444, 555]
N_FOLDS = 5

# 模型参数，按报告调优结果直接写死
lgb_params = {
    'learning_rate': 0.0531,
    'num_leaves': 73,
    'min_child_samples': 114,
    'subsample': 0.8481,
    'colsample_bytree': 0.7846,
    'reg_alpha': 0.5,
    'reg_lambda': 2.0,
    'n_estimators': 1000,
    'data_sample_strategy': 'goss',
    'verbosity': -1,
    'n_jobs': -1,
}

xgb_params = {
    'learning_rate': 0.047,
    'max_depth': 6,
    'min_child_weight': 15,
    'subsample': 0.82,
    'colsample_bytree': 0.76,
    'reg_alpha': 0.3,
    'reg_lambda': 1.5,
    'n_estimators': 1200,
    'verbosity': 0,
    'n_jobs': -1,
    'eval_metric': 'auc'
}

cat_params = {
    'learning_rate': 0.062,
    'depth': 7,
    'l2_leaf_reg': 3.0,
    'subsample': 0.85,
    'iterations': 1000,
    'verbose': 0,
    'eval_metric': 'AUC'
}

# ===================== 读数据 + 全局统计 =====================
train = pd.read_csv(os.path.join(DATA_PATH, "train.csv"))
test = pd.read_csv(os.path.join(DATA_PATH, "test.csv"))
sub = pd.read_csv(os.path.join(DATA_PATH, "sample_submission.csv"))

# 全量算无标签的客观统计量，绝对不碰目标
all_data = pd.concat([train, test], ignore_index=True)
race_total_laps = all_data.groupby('Race')['LapNumber'].max().to_dict()
race_avg_lap = all_data.groupby('Race')['LapTime (s)'].mean().to_dict()
comp_avg_life = all_data.groupby('Compound')['TyreLife'].mean().to_dict()

# ===================== 特征工程 =====================
def make_features(df):
    df = df.copy()
    
    # 预处理：缺省值 + 3sigma截断异常值
    num_cols = df.select_dtypes('number').columns
    cat_cols = df.select_dtypes('object').columns
    df[num_cols] = df[num_cols].fillna(0)
    df[cat_cols] = df[cat_cols].fillna('__NA__')
    
    for col in ['LapTime (s)', 'LapTime_Delta', 'Cumulative_Degradation']:
        mu, sigma = df[col].mean(), df[col].std()
        df[col] = np.clip(df[col], mu - 3*sigma, mu + 3*sigma)
    
    # 1. 基础衍生特征
    # 比赛进度
    df['TotalLaps'] = df['Race'].map(race_total_laps)
    df['Laps_Remaining'] = df['TotalLaps'] - df['LapNumber']
    df['RaceProgress'] = df['LapNumber'] / df['TotalLaps']
    
    # 轮胎磨损系列
    df['TyreWearRate'] = df['Cumulative_Degradation'] / (df['TyreLife'] + 1)
    df['TyreLife_Relative'] = df['TyreLife'] / df['Compound'].map(comp_avg_life)
    df['TyreLife_Remain'] = df['TyreLife'] * (1 - df['RaceProgress'])
    
    # 位置与圈速相对值
    df['Pos_Change_Rate'] = df['Position_Change'] / (df['LapNumber'] + 1)
    df['LapTime_Relative'] = df['LapTime (s)'] / df['Race'].map(race_avg_lap)
    
    # 2. 业务分箱，给树模型提供切分点
    df['TyreLife_Bin'] = pd.cut(df['TyreLife'], bins=[-1, 5, 10, 15, 20, 25, 30, 999], labels=False)
    df['RaceStage'] = pd.cut(df['RaceProgress'], bins=[-0.01, 0.3, 0.7, 1.01], labels=False)
    df['Pos_Bin'] = pd.cut(df['Position'], bins=[0, 5, 10, 15, 20, 99], labels=False)
    
    # 3. 数字位特征，抓隐藏周期规律
    for col in ['LapTime (s)', 'TyreLife']:
        df[f'{col}_int'] = df[col].astype(int)
        df[f'{col}_dec1'] = (df[col] * 10).astype(int) % 10
    
    # 4. 频率编码
    for col in ['Driver', 'Compound', 'Race']:
        df[f'{col}_freq'] = df[col].map(df[col].value_counts(normalize=True))
    
    # 5. 聚合类特征（对应报告的自动特征，手写核心几个，不用featuretools避免噪声）
    df['Track_Comp_AvgLife'] = df.groupby(['Race', 'Compound'])['TyreLife'].transform('mean')
    df['Driver_Comp_MedianLap'] = df.groupby(['Driver', 'Compound'])['LapNumber'].transform('median')
    df['Stage_Pos_Mean'] = df.groupby(['Race', 'RaceStage'])['Position'].transform('mean')
    
    # 类别特征基础编码
    for col in cat_cols:
        le = LabelEncoder()
        df[col] = le.fit_transform(df[col].astype(str))
    
    return df

# ===================== 工具：折内目标编码 =====================
def target_encode(tr_df, val_df, cols, target, smooth=10):
    tr_df = tr_df.copy()
    val_df = val_df.copy()
    global_mean = target.mean()
    
    for col in cols:
        tmp = pd.DataFrame({'cat': tr_df[col], 'y': target})
        agg = tmp.groupby('cat')['y'].agg(['mean', 'count'])
        agg['smooth'] = (agg['mean']*agg['count'] + global_mean*smooth) / (agg['count'] + smooth)
        mapping = agg['smooth'].to_dict()
        
        tr_df[f'{col}_te'] = tr_df[col].map(mapping)
        val_df[f'{col}_te'] = val_df[col].map(mapping).fillna(global_mean)
    
    return tr_df, val_df

# ===================== 单模型交叉验证训练 =====================
def cv_train(model_cls, params, X, y, X_test, seed):
    skf = StratifiedKFold(n_splits=N_FOLDS, shuffle=True, random_state=seed)
    oof = np.zeros(len(y))
    test_pred = np.zeros(len(X_test))
    te_cols = ['Driver', 'Compound', 'Race', 'TyreLife_Bin', 'RaceStage']
    
    for fold, (tr_idx, val_idx) in enumerate(skf.split(X, y)):
        X_tr, X_val = X.iloc[tr_idx].copy(), X.iloc[val_idx].copy()
        y_tr, y_val = y[tr_idx], y[val_idx]
        
        # 折内做目标编码，彻底防泄露
        X_tr, X_val = target_encode(X_tr, X_val, te_cols, y_tr)
        _, X_te = target_encode(X_tr, X_test.copy(), te_cols, y_tr)
        
        p = params.copy()
        p['random_state'] = seed
        model = model_cls(**p)
        
        # 不同模型早停写法适配
        if model_cls == LGBMClassifier:
            model.fit(X_tr, y_tr, eval_set=[(X_val, y_val)], eval_metric='auc',
                      callbacks=[early_stopping(50, verbose=False)])
        elif model_cls == XGBClassifier:
            model.fit(X_tr, y_tr, eval_set=[(X_val, y_val)], verbose=False, early_stopping_rounds=50)
        else:
            model.fit(X_tr, y_tr, eval_set=(X_val, y_val), early_stopping_rounds=50, verbose=False)
        
        oof[val_idx] = model.predict_proba(X_val)[:, 1]
        test_pred += model.predict_proba(X_te)[:, 1] / N_FOLDS
    
    return oof, test_pred

# ===================== 伪标签增强 =====================
def add_pseudo(X_train, y_train, X_test, pred, threshold=0.95):
    # 只取置信度极高的样本当伪标签
    pos_mask = pred > threshold
    neg_mask = pred < (1 - threshold)
    
    pseudo_X = X_test[pos_mask | neg_mask].reset_index(drop=True)
    pseudo_y = (pred[pos_mask | neg_mask] > 0.5).astype(int)
    
    X_new = pd.concat([X_train, pseudo_X], ignore_index=True)
    y_new = np.concatenate([y_train, pseudo_y])
    
    print(f"  加入伪标签：正样本{pos_mask.sum()}，负样本{neg_mask.sum()}")
    return X_new, y_new

# ===================== 主流程 =====================
if __name__ == '__main__':
    print("开始跑最终版...")
    
    # 生成特征
    print("生成特征...")
    train_feat = make_features(train)
    test_feat = make_features(test)
    
    drop_cols = ['id', 'PitNextLap']
    feats = [c for c in train_feat.columns if c not in drop_cols]
    X = train_feat[feats].reset_index(drop=True)
    y = train['PitNextLap'].values
    X_te = test_feat[feats].reset_index(drop=True)
    
    print(f"特征维度：{X.shape}")
    
    # 第一步：先训一版LGB拿伪标签
    print("\n第一步：训练基础模型生成伪标签")
    base_oof, base_test = cv_train(LGBMClassifier, lgb_params, X, y, X_te, seed=42)
    print(f"  基础模型OOF AUC: {roc_auc_score(y, base_oof):.6f}")
    
    X_aug, y_aug = add_pseudo(X, y, X_te, base_test)
    
    # 第二步：三模型多种子训练
    print("\n第二步：三模型多种子训练（含伪标签）")
    model_list = [
        (LGBMClassifier, lgb_params, 'LightGBM'),
        (XGBClassifier, xgb_params, 'XGBoost'),
        (CatBoostClassifier, cat_params, 'CatBoost')
    ]
    
    oof_all = []
    test_all = []
    
    for cls, params, name in model_list:
        print(f"\n训练 {name}：")
        oof_mean = np.zeros(len(y))
        test_mean = np.zeros(len(X_te))
        
        for seed in SEEDS:
            oof, test_pred = cv_train(cls, params, X_aug, y_aug, X_te, seed)
            # 只取原训练集部分算分，伪标签部分不计入指标
            oof_clean = oof[:len(y)]
            oof_mean += oof_clean / len(SEEDS)
            test_mean += test_pred / len(SEEDS)
            print(f"  seed {seed}: {roc_auc_score(y, oof_clean):.6f}")
        
        oof_all.append(oof_mean)
        test_all.append(test_mean)
        print(f"  {name} 平均: {roc_auc_score(y, oof_mean):.6f}")
    
    # 第三步：Stacking融合
    print("\n第三步：Stacking两层融合")
    meta_train = np.column_stack(oof_all)
    meta_test = np.column_stack(test_all)
    
    meta_model = LogisticRegression(random_state=42)
    meta_model.fit(meta_train, y)
    
    final_oof = meta_model.predict_proba(meta_train)[:, 1]
    final_pred = meta_model.predict_proba(meta_test)[:, 1]
    
    print(f"  融合后OOF AUC: {roc_auc_score(y, final_oof):.6f}")
    
    # 保存提交
    final_pred = np.clip(final_pred, 0.001, 0.999)
    sub['PitNextLap'] = final_pred
    save_file = os.path.join(SAVE_PATH, "f1_pit_final.csv")
    sub.to_csv(save_file, index=False)
    
    print(f"\n跑完了，文件存在：{save_file}"）
