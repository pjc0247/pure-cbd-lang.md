# pure-cbd-lang.md
Pure Componented Based Programming Language Specification<br>
Based on [my old gist](https://gist.github.com/pjc0247/ac68949afab83b477d0c7840a18170ea), added some modifications and translations.<br>
<br>
Simillar with Unity's __[ECS](https://unity3d.com/kr/learn/tutorials/topics/scripting/introduction-ecs)__.<br>
<br>
It does not contains any implementations.

Specifications
====
Component
----
컴포넌트는 하나의 독립 개채로 존재할 수 있으면서, 동시에 합쳐질 수 잇는 단위입니다.<br><br>

컴포넌트는 다른 컴포넌트를 포함할 수 있습니다.<br>
포함된다는것은 부모-자식 관계가 아니며 계층 구조상 동일한 위치에 속합니다.
```
Component MyGameObject
  : Renderer as renderer
  
  def foo
    renderer.texture = nil
  end
end
```

Protocol
----
__Protocol__ is a set of components.<br>
프로토콜 안에 포함되는 컴포넌트는 중복되는 프로퍼티, 메소드가 포함되지 않음이 보장되어야 합니다.
```
Component GameObject
  int hp
  string name
end
Component Renderer
  float opacity
end

Protocol Object2D
  : GameObject,
    Renderer
```

__Usage__
```
Component Enemy
  : Object2D as obj
  
  def damage n
    obj.hp -= n
    
    if obj.hp < 0
      obj.opacity = 0
    end
  end
end
```

Hierarchy
----
Supports __Hierarchy__ in language level.<br>
Use `addChild` keyword to append child component.
```
addChild HpStatusBar
```

Hierarchy Variable
----
계층 변수는 부모에서 자식으로 넘어올 때 어떻게 계산될지를 지정할 수 있습니다.
```
Component GameObject
  // Positions should be added
  [Add] float x = 0, y = 0
  // Opacity should be multiplied
  [Mul] float opacity = 1.0f
end
```
use `g` syntax to retrive root values.
```
// this code always fetches root value.
int globalSpeed = g::speed
```
만약 부모 컴포넌트에 존재하는 프로퍼티를 같은 이름으로 재정의하는것으로 계층 변수를 풀어버릴 수 있습니다.
```
// 부모 투명도가 몇이든 항상 보임
Component AlwaysVisible
  float opacity = 1.0f
  
  // 또는 [Pure]를 붙이는것으로 명시적 지정할 수 있습니다.
  // 이는 타 언어에서 명시적인 override 키워드와 같은 기능을 지닙니다.
  // [Pure] float opacity = 1.0f
end
```

Animated Variable
----
프로퍼티를 자동으로 애니메이트 시킬 수 있습니다. <br>
속도를 지정하지 않으면 자동으로 설정값(계층 변수로 된)으로 동작합니다.
```
opacity -> 0
```
애니메이션 도중에 새로운 값 대입이 발생하면 기존 애니메이션은 중지됩니다.
```
// Stops all previous animations and assign
opacity = 255
```

<br>
[FxUnity](https://github.com/pjc0247/FxUnity)

Messaging
----
In-process pubsub messaging is provided by syntax level.
```
Component UIButton
  // Subscriber starts with `on` keyword.
  // These methods' return type is always 'void'
  on :click e
    // mouse click position
    e.x, e.y
  end
end
```
Use `publish` keyword to publish messages.
```
Component MouseInputDriver
  /* .... */
    publish :click {x => 1, y => 1}
  /* .... */
end
```
Use `publish_local` to publish messages in local scope. This will be delivered in a local object.
```
publish_local :click {x => 1, y => 1}
```

Injectable
----
`Injectable`은 다른 컴포넌트에 자동으로 주입되는 서브-컴포넌트입니다. 이 기능은 다른 언어/프레임워크등에서의 AOP에 대응합니다.<br>
다른 AOP와의 차이점은 주입 받는 쪽에서 주입할 컴포넌트를 지정하는것이 아닌, 주입할 쪽에서 주입 받는 타겟 컴포넌트를 지정할 수 있습니다.
```
// 이름이 UI로 시작하는 컴포넌트에 자동으로 주입됩니다.
// 이러한 기능은 컴포넌트를 이름으로써 명확하게 분류할 수 있을 때 유용합니다.
Injectable UIFade("UI*")
    : Renderer as renderer
    
    on :pause
        renderer.opacity -> 0
    end
end

Component UIMessageBox 
    : Renderer as renderer
    
    on :draw
    end
end
```

Threading model
----
메세징에 의한 멀티 스레딩과, 코루틴에 의한 멀티 스레딩 두가지를 지원합니다.<br>
둘다 락이 없으며, ㅁㄴㅇㄻㄴㅇㄹ
<br><br>
__Background Task__<br>

```
Component UINotice
  background_task fetch
    /* load_notices_from_www */
    publish_local :done
  end
  
  def init
    fetch
  end
  on :done
    opacity -> 255
  end
end
```
__Foreground Task__<br?

```
task http_get uri
  /* ..... */
end

def on_click
  body = http_get "www.naver.com"
end
```

`commit` 블럭은 락없는 동기화를 가능하도록 해줍니다.<br>
이는 오브젝트에 대한 쓰기락을 획득하는 방식이 아닌, 로컬 메세지 큐 기반으로 동작합니다. 
```
Component UIProfile 
  : User as user
  
  task update_nickname
    profile = load_user_profile_from_www
  
    commit
      // This codeblock will be ran on main-thread.
      // 이 블럭은 안전한 함수여야 합니다.
      user.profile = profile
    end
  end
end
```

Subscribers with `on` always be executed on main-thread regardless the thread which `publish` was executed.
```
on :hi
  // Always executed in foreground thread
end
```
Subscribers with `background_on` always be executed on background-thread.
```
background_on :hi
  // Always executed in background thread
end
```

Safe function
----
안전한 함수는 익셉션이 발생하지 않는 함수 또는 코드 블럭입니다.
* 정수형의 나누기는 금지됩니다.
* 안전하지 않은 함수의 호출은 금지됩니다.

```
[Safe]
def foo a, b
  return a + b
end
```
