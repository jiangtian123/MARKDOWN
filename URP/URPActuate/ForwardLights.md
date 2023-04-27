# ForwardLights
## 数据
| 类型               | 名称                                    | 备注                         |
| :----------------- | :-------------------------------------- | :--------------------------- |
| int                | m_AdditionalLightsBufferId              |                              |
| int                | m_AdditionalLightsIndicesId             |                              |
| MixedLightingSetup | m_MixedLightingSetup                    |                              |
| Vector4[]          | m_AdditionalLightPositions              |                              |
|                    | m_AdditionalLightColors                 |                              |
|                    | m_AdditionalLightAttenuations           |                              |
|                    | m_AdditionalLightSpotDirections         |                              |
|                    | m_AdditionalLightOcclusionProbeChannels |                              |
| float[]            | m_AdditionalLightsLayerMasks            |                              |
| bool               | m_UseStructuredBuffer                   | 这个buffer是计算着色器使用的 |
| bool               | m_UseClusteredRendering                 |
| int                | m_DirectionalLightCount                 |
 