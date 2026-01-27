---
title: FSM\UNITY
date: 2026-01-27 16:41:07
tags:
---
# UNITY|有限状态机|FSM

此文章为我在学习unity‘时记录，方便以后快速对比使用

**首先是为什么要用到有限状态机（FSM），该如何使用？**

想一想你自己之前是如何实现玩家控制的？直接如下吗？

     if(Input.GetKeyDown(KeyCode.Space))
        {
            //跳跃
        }

确实，这样是一件十分简单有效的方法。然而当你在跳跃的时候按下前进/突进、攻击等类似按键后，玩家所操作的角色应该如何应对呢？可以继续沿着水平操作在空中移动？是否可以攻击、攻击形式又是如何呢？

当然，你可以再加入若干的if-else来应对具体的情况。不过这不可避免的考虑一切状态、跑步跳跃攻击甚至ui操作的时候你都要百不厌其烦的if-else。尤其当一个游戏项目大起来后，if里嵌套了更多的if简直让人头晕、后期优化修复的时候也是令人头大。

介于此、我们要引入一个优秀的设计：

***有限状态机(FSM)***

现在对其进行原理解释：将对象的各种行为以状态的形式进行区分，通过状态机统筹这些状态，一般一个状态机负责一类状态，当进入某状态时，该状态机所负责的部分全权交给此状态的内容。之前状态被替换，效果消失。在状态操作中可以加入判断、通知状态机变化状态。

1.状态机

首先是创立好每个状态机的基本规范（也就是一个状态机的抽象类），状态机应该记录一个当前状态（currentstate），负责改变状态（ChangeState、记住要传入一个状态的参数作为新的状态）和处理一些输入更新。基本如下

    namespace StateMachine
    {
        public abstract class StateMachine
        {
            protected IState currentState;

            public void ChangeState(IState newState)
            {
                currentState?.Exit();
                currentState = newState;
                currentState.Enter();
            }

            public void HandleInput()
            {
                currentState?.Handleinput();
            }
            public void Update()
            {
                currentState?.Update();
            }
            public void PhysicsUpdate()
            {
                currentState?.PhysicsUpdate();
            }
        }
    }

接着我们可以通过继承来创造具体的状态机，比如移动状态机MoveStateMachine

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class MoveStateMachine : StateMachine.StateMachine
    {
      public Player player;//这里接收玩家
      public WalkState walkState;//存好一个状态
      public MoveStateMachine(Player _player)//当被实例化时的初始化，要先赋值玩家、同时实例化一个状态
      {
          player = _player;
          walkState = new WalkState(this);//这里的this对应的要在状态的构造函数里进行重载修改、让状态知道自己被状态机使用了
      }
    }
    ﻿

2.状态与基类状态

首先是状态，我以地面移动为例，姑且分为Walk（行走）与Run（奔跑）两个状态。在这两个状态被制造之前，我们应该对状态进行规范。也就是创立基类状态ISate。

我们规定状态至少有这么几个方法：进入状态时、对input反映、update更新内容、固定帧更新内容、退出状态时。如下建立一个接口

     using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
     
     public interface IState
     {

      public void Enter();
        
      public void Handleinput();
        
      public void Update();
        
      public void PhysicsUpdate();
        
      public void Exit();
     }

    ﻿

有了基本状态的规范后我们就创立具体的状态叭\~\~

首先walk和run都可以归纳为MoveState。都有共同的移动逻辑，只有部分区别。那就先建立一个 MoveState作为walk状态的基类 MoveState 。

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class MoveState : IState
    {
        public MoveStateMachine moveStateMachine;//要拿到是哪一个状态机取的自己，之后得通过状态机对外界交接
        protected Vector2 movementInput;
        protected float baseSpeed =1f;
        public float speedModifier = ALL_Data.PlayerWalkSpeedModify;// 此处自己定义一个速度修正值我把这个存入一个单例里了（自己随意），本例里不在赘述

        public MoveState(MoveStateMachine _moveStateMachine)
        {//构造器，当这个状态被实例化时要拿到状态机
            moveStateMachine = _moveStateMachine;
        }
        
        #region Istate Methods
    //Istate里的基本函数
        public virtual void Enter()
        {
            Debug.Log("State"+GetType().Name);
        }

        public virtual void Handleinput()
        {
            ReadMovementInput();//读取输入
        }

        public virtual void Update()
        {
            Move();//实际移动
        }

        public virtual void PhysicsUpdate()
        {
            
        }

        public virtual void Exit()
        {
            
        }
        
        #endregion
        
        #region Main Methods
    //此处是在该状态里写的具体主要函数
        public  void ReadMovementInput()
        {
            movementInput = moveStateMachine.player.PlayerInput.playerActions.Move.ReadValue<Vector2>();
        }
        private void Move()
        {
            if (movementInput == Vector2.zero||speedModifier ==0f)
            {
                return;
            }

            Vector2 movementDirection = GetMoveMentDirection();
                
            moveStateMachine.player.rb.MovePosition(movementInput*movementDirection+moveStateMachine.player.rb.position);
            moveStateMachine.player.rb.velocity = new Vector3 (0f,0f,0f);
            Debug.Log("xy:"+movementInput+"v"+moveStateMachine.player.rb.velocity);
            
        }

           

        #endregion

        #region reusable Methods

        private float GetMovementSpeed()
        {
            return baseSpeed * speedModifier*Time.deltaTime;
            //TOLOOK delttime
        }
        private Vector2 GetMoveMentDirection()
        {
            float movementSpeed = GetMovementSpeed();
            return new Vector2(movementSpeed,movementSpeed);
        }
        #endregion
    }
    ﻿

有了MoveState后我们就可以创造WalkState了，由于没有什么需要更改的就能基本移动，所以就不写什么函数了

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class WalkState : MoveState
    {
        public WalkState(MoveStateMachine _moveStateMachine) : base(_moveStateMachine)
        {//要拿到是哪一个状态机取的自己，之后得通过状态机对外界交接
        }
        
    }
    ﻿

至此有限状态机基本实现了，

现在来回顾一下，要有状态（基类与派生类）、状态机（基类与派生类）。状态机相当于一个容器存储不同的行为逻辑、状态则是具体的逻辑。所有的移动方式都要让移动状态机管理、避免不同状态同时存在的冲突。改变状态是调用状态机的ChangeState函数。这样就把诺干个if化解开啦！
