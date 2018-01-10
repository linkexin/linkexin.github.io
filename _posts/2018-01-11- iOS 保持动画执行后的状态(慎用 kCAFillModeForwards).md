---
title: iOS 保持动画执行后的状态（慎用 kCAFillModeForwards）
---

### 问题的提出

使用 CAAnimations 实现动画在日常开发中很常见
比如说我们想让 view 实现一个透明度和弹性动画的组合动画，下面是通常的做法：

```
//放大
CASpringAnimation *expandAnimation = [CASpringAnimation animationWithKeyPath:@"transform.scale"];
expandAnimation.fromValue = [NSNumber numberWithFloat:0];
expandAnimation.toValue = [NSNumber numberWithFloat:1.0f];
expandAnimation.damping = 20;
expandAnimation.duration = expandAnimation.settlingDuration;
expandAnimation.stiffness = 380;
expandAnimation.mass = 1;
expandAnimation.initialVelocity = 0;
expandAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    
//透明度
CABasicAnimation *opacityAnimation = [CABasicAnimation animationWithKeyPath:@"opacity"];
opacityAnimation.duration = 0.2;
opacityAnimation.fromValue = [NSNumber numberWithFloat:0];
opacityAnimation.toValue = [NSNumber numberWithFloat:1.0f];
opacityAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    
CAAnimationGroup *groups = [CAAnimationGroup animation];
groups.animations = @[expandAnimation, opacityAnimation];
groups.duration = expandAnimation.settlingDuration;

[view.layer addAnimation:groups forKey:@"group"];
```

但是这么执行会发现 view 在结束动画后，就会瞬间回到动画前的状态。
如果我们想要 view 保持动画执行后的状态，我们通常将 fillMode 设置为 kCAFillModeForwards，再将 removedOnCompletion 设置为 NO：

```
groups.removedOnCompletion = NO;
groups.fillMode = kCAFillModeForwards;
```
这么设置确实会达到我们想要的效果，因为我们确实让动画效果永远的存在了，但实际上，这种方法只是隐藏了一些现象，当我们真的去查看 layer 的时候，会发现 layer 的 透明度 和 大小 的值是动画执行之前的值，而且只要你后续再对这个 view 的这两个属性值设置其他的动画，就会出现问题。

### 更好的解决方法
其实动画的 fromValue 和 toValue 是可以相互转换的，为了避免上面提到的问题，我们要将 layer 的状态设置为动画执行后的状态，然后通过设置 fromValue 来实现动画：

```
//放大
CASpringAnimation *expandAnimation = [CASpringAnimation animationWithKeyPath:@"transform.scale"];
expandAnimation.fromValue = [NSNumber numberWithFloat:0];
expandAnimation.damping = 20;
expandAnimation.duration = expandAnimation.settlingDuration;
expandAnimation.stiffness = 380;
expandAnimation.mass = 1;
expandAnimation.initialVelocity = 0;
expandAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    
//透明度
CABasicAnimation *opacityAnimation = [CABasicAnimation animationWithKeyPath:@"opacity"];
opacityAnimation.duration = 0.2;
opacityAnimation.fromValue = [NSNumber numberWithFloat:0];
opacityAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    
CAAnimationGroup *groups = [CAAnimationGroup animation];
groups.animations = @[expandAnimation, opacityAnimation];
groups.duration = expandAnimation.settlingDuration;

view.layer.opacity = 1;
[view.layer setBounds:CGRectMake(0, 0, 48, 48)];
[view.layer addAnimation:groups forKey:@"group"];
```






