# 环境光照
## IBL
IBL 表示的是环境光贴图。有两种表现方式

1. CubeMap
2. SphereMap
### 计算Shading
给定一张IBL，在不考虑遮挡的情况下，计算Shading。

- 不考虑遮挡的渲染方程
  
  **L0（p,$\omega_0$）= $\int_ΩL_i(p,\omega_i)f_r(p,\omega_i,\omega_0)cos\theta_id\omega_i$**


