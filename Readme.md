# 각 script에서 중요한 내용 정리
## 0. Unity개요 & keywords
---------------
* **[HideInInspector]:** public variable이지만 Inspector창에는 보이지 않도록한다.
* 사용한 Event함수의 실행 순서 (참고 : https://docs.unity3d.com/Manual/ExecutionOrder.html)
> **처음 Scene이 Load 되었을 때**  
>> scene이 load 되고 scene의 object들에 대해서 각각 호출되는 함수들
>* **Awake :** prefab이 instance화 된 직후에 실행되는 event 함수이다. **Start** 보다 먼저 실행된다.  
이 프로젝트에서는 Main Camera의 **Awake**에서 gameManager의 singleton를 initiate했다.
>* **OnEnable :** 오브젝트가 활성화 된 직후에 호출된다.  
이 프로젝트에서는  object가 eventManager에게 subscribe해야 할 함수가 있을 때 **OnEnable**에서 subscribe했다.
>```cs
>void OnEnable()
>    {
>        //scnenManager(eventmanager)에 있는 sceneLoaded(event)가 일어 날 때 OnLevelFIsnishedLoading함수가 실행 되도록 subscribe.
>        SceneManager.sceneLoaded += OnLevelFinishedLoading;
>    }
>```
---------------
>**첫 Frame이 Update 되기 직전**  
>* **Start :** 이 프로젝트에서는 Player, Enemy등 하위 object들에 대한 초기화를 **Start**에서 했다. 
---------------
>**Frame Update**
>* **FixedUpdate :** 사용x
>* **Update :** Frame당 한 번 호출 되는 함수.  
이 프로젝트 에서는 유저의 input과 animation 처리를 **Update**에서 했다. 
>* **LateUpdate :** 사용x
---------------
>**Coroutine**
>>Coroutine은 Update 함수가 return된 후 실행된다. 일반적으로 Coroutine이란 yield를 yieldInstruction이 끝날 때 까지 기다리는 함수를 말한다. 
>* **yield :** Update가 끝나고 실행된다.
>* **yield WaitForSeceond(sec) :** Update가 끝나고, sec후 실행된다.
>* **yield StartCoroutine :** Coroutine이 끝나고 실행된다.
---------------
>**Obejct is Destroyed**
>* **OnDestroyed :** object의 마지막 Frame이 update된 후에 호출 된다.
---------------
>**When Quitting**
>>scene의 모든 object에게 호출 된다.
>* **OnDisable :** 이 프로젝트에서는 enable때 subscribe했던 function을 **OnDisable**에서 desubscribe했다. parameter 저장?
## 1. GameManager.cs  
---------------
게임 전반을 관리하는 script.  
* 아래 코드처럼 하나의 instance만 존재하도록 **singleton**으로 만든다.
* 또한 gameManager가 새로운     scene이 불렸을 때에도 새로 생성되지 않도록  DontDestroyOnLoad(gameObject)를 호출했다.
```cs
public static GameManager instance = null;

void Awake () {
    if (instance == null)
       instance = this;
    else if (instance != this)
        Destroy(gameObject);    

    DontDestroyOnLoad(gameObject);
}
```
* Update에서 frame마다 state를 판단하고 적절한 경우에 Coroutine을 이용해서 적을 움직인다. 일반적으로 Coroutine은 여러 frame에 걸쳐서 (혹은 frame에 관계없이) 계산을 실행하고 싶을 때 사용되는데 이 경우에는 적들을 순차적으로 이동시키는 작업을 위해서 사용되었다.  
* **StartCoutine**은 IEnumerator의 MoveNext가 false일 때 까지 routine을 돌리는 함수이다.
```cs
void Update () {
        if (playersTurn || enemiesMoving || doingSetup)
            return;
        StartCoroutine(MoveEnemies());
	}

IEnumerator MoveEnemies()
    {
        enemiesMoving = true;
        yield return new WaitForSeconds(turnDelay);
        if (enemies.Count == 0)
        {
            yield return new WaitForSeconds(turnDelay);
        }

        for(int i = 0; i < enemies.Count; i++)
        {
            enemies[i].MoveEnemy();
            yield return new WaitForSeconds(enemies[i].moveTime);
        }

        playersTurn = true;
        enemiesMoving = false;
    }
```
## 2. MovingObject.cs
--------------------------------
* 게임에서 움직이는 object인 enemy와 player를 상속을 이용해서 구현했다. 
* object에 붙은 **abstract**는 이 object가 반드시 상속되어서 구현되어야만 한다는 것을 의미한다. 함수 앞에 붙은 **abstract**는 이 함수가 반드시 derived class에서 구현되어야 한다는 것을 의미한다. 
* **virtual**은 derived class에서 재정의 될 수 있는 멤버를 의미한다.
* **protected**는 derived class를 통해서만 접근 할 수 있는 멤버를 의미한다. 이 프로젝트에서 movingObject는 abstract class이기 때문에 인스턴스화 될 수 없고, 내부 함수들에 직접 접근하는 경우는 없다.
```cs
public abstract class MovingObject : MonoBehaviour {

    ...

    protected virtual void Start () {
        ...
    }

    protected abstract void OnCantMove<T>(T component)
        where T : Component;
}
```
## 3. Enemy.cs  
--------------------------------
* 구현, 재정의하는 함수에는 **override** 키워드를 사용한다.
* override된 함수에서는 Type T를 안다고 생각하고 사용해야 하는데 이 때 **as** 키워드를 이용해서 casting한 후에 사용한다. **as**는 ()형변환과 다르게 형변환이 불가능 한 경우 예외를 발생시키지 않고 **null**을 반환한다. **as**는  reference type에만 사용 가능하다.
* 참고로 **is**는 형변환이 가능한 경우 **true**를 불가능한 경우 **false**를 반환한다.
```cs
protected override void OnCantMove<T>(T component)
    {
        Player hitplayer = component as Player;
        
        ...
    }
```
* **Reference type vs Value type**  
c#에는 두가지의 type 종류가 존재한다. 기본 type과 사용자 정의 구조체는 value type이고 string, 클래스, 배열의 instance등은 reference type이다. value는 스택에 저장되어 block을 벗어나면 삭제되고 reference는 힙에 저장되어 C#의 GB가 필요 없다고 판단 할 때에 삭제된다. 
* **Boxing , Unboxing**  
boxing이란 value를 reference로 형변환 하는 것을 의미한다. 
```cs
int i = 67;                              // i is a value type
object o = i;                            // i is boxed
System.Console.WriteLine(i.ToString());  // i is boxed
i = (int)o;                              // o is unboxed
```
* **ref**, **out** 키워드와 성능 문제  
함수에 매개변수로 value를 전달하면 스택에 value의 복사본을 만들어야 한다. 그럴 때 **ref**를 사용해서 참조를 전달하면 오버헤드를 줄일 수 있다(c++에서의 포인터와 같다).  
**out**은 ref와 유사하지만 함수내에서 반드시 매개변수에 값을 할당 해야 한다.

## 4. Player.cs  
--------------------------------
* input을 update에서 받아서 처리했다. 이 때 모바일 환경을 위해서 #if #else 구문을 사용했다. 
각 환경의 Input에 대해서는 UnityEngine.Input의 API를 참고
```cs
#if UNITY_STANDALONE || UNITY_WEBPLAYER


        horizontal = (int)Input.GetAxisRaw("Horizontal");
        vertical = (int)Input.GetAxisRaw("Vertical");

        ...
#elif UNITY_IOS || UNITY_ANDROID || UNITY_WP8 || UNITY_IPHONE

            Touch myTouch = Input.touches[0];

            ...
#endif

```


# 안드로이드 빌드 환경 구축하기
http://webnautes.tistory.com/1006 참고