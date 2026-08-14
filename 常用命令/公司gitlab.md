access token：6nQJN2xtBpPtfuGmuya9

AVd36vk$
您好，明天在提供宠物面部生物特征识别接口后，我们会紧接着准备宠语翻译，在这之前想了解下您们的测评需求，我先介绍下目前我们模型的输入和输出规则：
宠语翻译：需要先选择与狗对话（狗语），还是与猫对话（猫语），我们也有猫犬叫声二分类模型
1. 输入：翻译宠语可以输入纯音频、视频（视频翻译会准确一些）；翻译人语只能输入音频
2. 输出：翻译人语输出宠物音频；翻译宠语输出文本
3. 一次完整会话包含2次，一来一回，回复会参考上下文来翻译，所以有个相关度
情绪识别：
4. 输入：宠语视频、音频
5. 输出：情绪标签、互动建议
6.   3. 再进 diskpart 重新来：

  select vdisk file="D:\wsl\ubuntu\ext4.vhdx"
  attach vdisk readonly
  compact vdisk
  detach vdisk
  exit