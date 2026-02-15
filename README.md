# Unity_PortFolio
## 📋 프로젝트 개요
### 프로젝트 소개
**projectlup-v2025**는 6개의 게임 팀으로 나뉘어 협업한 다장르 통합 Unity 프로젝트로 프레임워크 개발자로 참여하여 공통 시스템을 개발했습니다.
5개의 게임(로그라이크, 덱 전략, 생산/건설/강화, 익스트랙션 슈터, 슈팅)이 하나의 프로젝트 안에서 독립적으로 동작하면서도 공통 프레임워크를 공유하는 구조입니다.

### 프로젝트 정보
- **프로젝트 저장소**: [projectlup-v2025](https://github.com/project-lup/projectlup-v2025)
- **개발 엔진**: Unity Engine 6000.0.62f
- **개발 언어**: C#
- **팀 구성**: 프로그래머 13명
- **담당 역할**: 프레임워크 개발
- **플랫폼**: PC

### 협업 도구
- **버전 관리**: Fork
- **문서화**: Notion

## 🎯 담당 업무
1. [**Google Sheets 데이터 연동 시스템**](#1-adapter-패턴과-리플렉션을-이용한-google-sheets-데이터-연동-시스템) - Adapter 패턴 기반 외부 데이터 파이프라인
2. [**Stage 전환 관리 시스템**](#2-코루틴-기반-비동기-stage-전환-시스템) - 코루틴 기반 비동기 전환 파이프라인과 Template Method 패턴의 Stage 생명주기
3. [**공용 인벤토리 시스템**](#3-커스텀-필드-기반-확장-가능한-공용-인벤토리-시스템) - 5개 게임이 공유하는 확장 가능한 아이템 관리
4. [**Pull Request 관리**](#4-pull-request-관리) - 13명 팀원의 코드 리뷰 및 머지 관리

## 💡 핵심 구현 내용
### 1. Adapter 패턴과 리플렉션을 이용한 Google Sheets 데이터 연동 시스템

#### 데이터 파이프라인 구조
```
Google Sheets (기획자/팀원 수치 관리)
    ↓ CSV Export URL
UnityWebRequest 비동기 로드
    ↓ CSVDataSourceAdapter
CSVParser (RFC 4180 따옴표 처리)
    ↓ ColumnAttribute 리플렉션 매핑
BaseStaticDataLoader<T> (ScriptableObject)
    ↓ 에디터 "데이터 읽기" 버튼
List<T> DataList (직렬화된 데이터)
    ↓ Runtime
DataManager → ResourceManager (Resources.LoadAll)
```

#### 개요
13명의 팀원이 각자 다른 데이터 시트를 사용하는 환경에서, 에디터 인스펙터의 **버튼 클릭 한 번**으로 Google Sheets의 데이터를 ScriptableObject에 동기화하는 시스템입니다. **Adapter 패턴**으로 데이터 소스를 추상화하여 향후 다른 소스(JSON API, Firebase 등)로의 확장이 가능하도록 설계했습니다.

#### 기술적 특징
- **IDataSourceAdapter 인터페이스** :  향후 데이터베이스 등 다른 소스로의 전환을 고려하여 처음부터 Adapter 패턴으로 데이터 소스를 추상화했습니다. 소스마다 로드 방식(HTTP, DB 쿼리)과 파싱 형식(CSV, JSON)이 다르지만, 이 차이를 어댑터 내부에 캡슐화하면 BaseStaticDataLoader가 소스 종류를 알 필요 없이 동일한 인터페이스로 호출할 수 있습니다. 결과적으로 새로운 데이터 소스 추가 시 어댑터 클래스 하나만 구현하면 기존 로직 수정이 불필요합니다.
- **ColumnAttribute + 리플렉션 자동 매핑** : 팀원이 데이터 클래스 필드에 `[Column("시트_컬럼명")]` 어트리뷰트만 붙이면, CSVDataSourceAdapter가 리플렉션으로 헤더와 필드를 자동 매핑합니다. int, float, string, bool, Enum 등 다양한 타입을 지원합니다.
- **ICustomFieldSupport** : `[Column]`으로 매핑되지 않은 나머지 컬럼들을 `Dictionary<string, string>`에 자동 수집합니다. 기획자가 시트에 새 컬럼을 추가해도 코드 수정 없이 데이터가 보존됩니다.
- **에디터 커스텀 인스펙터** : `BaseStaticDataReaderEditor`에서 "데이터 읽기" 버튼을 제공합니다. 에디터 환경에서는 코루틴을 직접 실행할 수 없으므로, `async/await`로 `IEnumerator`를 래핑하는 `ProcessCoroutine` 메서드를 구현하여 에디터에서도 비동기 웹 요청이 동작하도록 했습니다.

<details>
<summary><b>🔍 코드 — IDataSourceAdapter & CSVDataSourceAdapter</b></summary>

```csharp
// 데이터 소스 추상화 인터페이스
public interface IDataSourceAdapter
{
    IEnumerator LoadData(string url, System.Action<string> onSuccess, System.Action<string> onError);
    List<T> ParseToObjects<T>(string rawData, int startRow) where T : new();
}

// CSV 어댑터 구현 — 리플렉션 기반 자동 매핑
public class CSVDataSourceAdapter : IDataSourceAdapter
{
    public IEnumerator LoadData(string url, System.Action<string> onSuccess, System.Action<string> onError)
    {
        UnityWebRequest www = UnityWebRequest.Get(url);
        yield return www.SendWebRequest();

        if (www.result != UnityWebRequest.Result.Success)
            onError?.Invoke(www.error);
        else
            onSuccess?.Invoke(www.downloadHandler.text);
    }

    private T ParseDataRow<T>(string[] values, Dictionary<string, int> headerMap) where T : new()
    {
        T instance = new T();
        FieldInfo[] fields = typeof(T).GetFields(BindingFlags.Public | BindingFlags.Instance);
        HashSet<string> mappedColumns = new HashSet<string>();

        foreach (FieldInfo field in fields)
        {
            ColumnAttribute columnAttr = field.GetCustomAttribute<ColumnAttribute>();
            if (columnAttr == null) continue;

            string headerName = columnAttr.HeaderName;
            if (!headerMap.ContainsKey(headerName)) continue;

            int columnIndex = headerMap[headerName];
            mappedColumns.Add(headerName);

            string value = values[columnIndex].Trim();
            SetFieldValue(field, instance, value);
        }

        // 매핑되지 않은 컬럼 자동 수집 (ICustomFieldSupport)
        if (instance is LUP.ICustomFieldSupport customFieldSupport)
        {
            foreach (var header in headerMap)
            {
                if (mappedColumns.Contains(header.Key)) continue;
                if (header.Value < values.Length)
                {
                    string value = values[header.Value]?.Trim();
                    if (!string.IsNullOrEmpty(value))
                        customFieldSupport.SetCustomField(header.Key, value);
                }
            }
        }

        return instance;
    }
}
```
</details>

<details>
<summary><b>🔍 코드 — ColumnAttribute & 에디터 비동기 래핑</b></summary>

```csharp
// 컬럼 매핑 어트리뷰트 — 필드에 붙이면 CSV 헤더와 자동 매핑
[AttributeUsage(AttributeTargets.Field, AllowMultiple = false)]
public class ColumnAttribute : Attribute
{
    public string HeaderName { get; private set; }
    public bool Required { get; set; }

    public ColumnAttribute(string headerName)
    {
        HeaderName = headerName;
        Required = true;
    }
}

// 에디터에서 코루틴을 실행할 수 없는 문제 해결
// async/await로 IEnumerator를 래핑하여 UnityWebRequest 대기 처리
#if UNITY_EDITOR
[CustomEditor(typeof(BaseStaticDataLoader), true)]
public class BaseStaticDataReaderEditor : Editor
{
    private async void LoadDataAsync()
    {
        IEnumerator coroutine = data.LoadSheet();
        await ProcessCoroutine(coroutine);
        EditorUtility.SetDirty(data);
        AssetDatabase.SaveAssets();
    }

    private async System.Threading.Tasks.Task ProcessCoroutine(IEnumerator coroutine)
    {
        while (coroutine.MoveNext())
        {
            object current = coroutine.Current;
            if (current is UnityEngine.Networking.UnityWebRequestAsyncOperation asyncOp)
            {
                while (!asyncOp.isDone)
                    await System.Threading.Tasks.Task.Delay(100);
            }
            else if (current is IEnumerator nestedCoroutine)
            {
                await ProcessCoroutine(nestedCoroutine); // 중첩 코루틴 재귀 처리
            }
            else
            {
                await System.Threading.Tasks.Task.Delay(10);
            }
        }
    }
}
#endif
```
</details>

---

### 2. 코루틴 기반 비동기 Stage 전환 시스템

#### Stage 전환 파이프라인 구조
```
StageManager.LoadStage(targetStageKind)
    ↓ IsValidTransition() — 전환 테이블 검증
StartCoroutine(TransitionCoroutine)
    ├─ OnStageExit()
    │   ├─ currentStage.OnStageExit() — SaveDatas() 호출
    │   └─ FadeOut() — CanvasGroup alpha 0→1
    ├─ SceneManager.LoadSceneAsync(sceneName) — 비동기 씬 로드
    ├─ FindFirstObjectByType<BaseStage>() — 새 Stage 인스턴스 탐색
    ├─ OnStageEnter()
    │   ├─ currentStage.OnStageEnter() — LoadResources() → GetDatas() → ItemManager.LoadAllItems()
    │   └─ FadeIn() — CanvasGroup alpha 1→0
    └─ 전환 완료
```

#### 개요
5개의 게임 모드가 각각 독립된 Unity Scene으로 존재하는 환경에서, **Stage 간 전환을 코루틴 기반 비동기 파이프라인으로 제어**하는 시스템입니다. `BaseStage` 추상 클래스가 Template Method 패턴으로 Stage 생명주기(Enter → Stay → Exit)를 정의하고, 각 게임 팀은 이를 상속받아 자신의 게임 로직만 구현합니다.

#### 기술적 특징
- **코루틴 기반 비동기 전환 파이프라인** : Stage 전환의 전체 흐름(Exit → Fade Out → 씬 로드 → Enter → Fade In)을 하나의 코루틴 체인으로 구성했습니다. `SceneManager.LoadSceneAsync()`로 씬을 비동기 로드하여 전환 중 프레임 드랍을 방지하고, `isTransitioning` 플래그로 전환 중 중복 호출을 차단합니다. Fade 연출은 런타임에 동적 생성되는 `CanvasGroup`(sortingOrder 999)으로 처리하며, `DontDestroyOnLoad`로 씬 전환 간 유지됩니다.
- **Template Method 패턴의 BaseStage 생명주기** : `BaseStage` 추상 클래스가 `OnStageEnter()` → `OnStageStay()` → `OnStageExit()`의 호출 순서를 제어하고, 각 단계에서 실행되는 구체 로직(`LoadResources()`, `GetDatas()`, `SaveDatas()`)을 추상 메서드로 위임합니다. 이 구조 덕분에 각 게임 팀은 프레임워크의 전환 흐름을 이해할 필요 없이 자신의 Stage 클래스에서 리소스 로딩과 데이터 처리만 구현하면 됩니다. 또한 `BaseStage`는 `DataManager`를 통한 런타임 데이터 저장/로드 유틸리티를 제공하여, 게임 팀이 데이터 영속성 로직을 직접 작성하지 않아도 됩니다.

<details>
<summary><b>🔍 코드 — StageManager 전환 파이프라인</b></summary>

```csharp
public class StageManager : Singleton<StageManager>
{
    // StageKind → SceneList 매핑 (ScriptableObject 기반)
    private Dictionary<Define.StageKind, SceneList> sceneNameMap = new Dictionary<Define.StageKind, SceneList>();
    private bool isTransitioning = false;

    // Stage 전환 요청 — 전환 테이블 검증 후 코루틴 시작
    public void LoadStage(Define.StageKind targetStageKind, int sceneindex = -1)
    {
        if (isTransitioning)
        {
            Debug.LogWarning("Already transitioning!");
            return;
        }

        if (!IsValidTransition(currentStageKind, targetStageKind))
        {
            Debug.LogError($"Invalid transition: {currentStageKind} → {targetStageKind}");
            return;
        }

        StartCoroutine(TransitionCoroutine(targetStageKind, sceneindex));
    }

    // 전환 코루틴 — Exit → Fade Out → 비동기 씬 로드 → Enter → Fade In
    private IEnumerator TransitionCoroutine(Define.StageKind targetStageKind, int sceneindex = -1)
    {
        isTransitioning = true;

        // 현재 Stage 퇴장 처리 (SaveDatas + FadeOut)
        yield return StartCoroutine(OnStageExit());

        // SceneList에서 씬 이름 조회
        string sceneName = (sceneindex == -1)
            ? sceneNameMap[targetStageKind].sceneNames[0]
            : sceneNameMap[targetStageKind].sceneNames[sceneindex];

        // 비동기 씬 로드
        AsyncOperation asyncLoad = SceneManager.LoadSceneAsync(sceneName);
        while (!asyncLoad.isDone)
        {
            yield return new WaitForSeconds(0.1f);
        }

        // 새 Stage 인스턴스 탐색
        currentStageInstance = FindFirstObjectByType<BaseStage>();
        while (currentStageInstance == null)
        {
            currentStageInstance = FindFirstObjectByType<BaseStage>();
            yield return new WaitForSeconds(0.1f);
        }

        // 새 Stage 진입 처리 (LoadResources + GetDatas + FadeIn)
        yield return StartCoroutine(OnStageEnter());

        currentStageKind = targetStageKind;
        isTransitioning = false;
    }
}
```
</details>

<details>
<summary><b>🔍 코드 — BaseStage Template Method 패턴</b></summary>

```csharp
public abstract class BaseStage : MonoBehaviour
{
    [ReadOnly] public Define.StageKind StageKind = Define.StageKind.Main;

    // Template Method — 프레임워크가 호출 순서를 제어
    // StageManager가 OnStageEnter()를 호출하면 LoadResources → GetDatas → ItemManager 순서로 실행
    public virtual IEnumerator OnStageEnter()
    {
        LoadResources();
        GetDatas();
        yield return ItemManager.Instance.LoadAllItems();
        yield return null;
    }

    public virtual IEnumerator OnStageStay()
    {
        yield return null;
    }

    // StageManager가 OnStageExit()를 호출하면 SaveDatas 실행
    public virtual IEnumerator OnStageExit()
    {
        SaveDatas();
        yield return null;
    }

    // 각 게임 팀이 구현하는 추상 메서드
    protected abstract void LoadResources();  // 게임별 리소스 로딩
    protected abstract void GetDatas();       // 게임별 데이터 주입
    protected abstract void SaveDatas();      // 게임별 데이터 저장

    // 데이터 접근 유틸리티 — DataManager 연동
    protected void SaveRuntimeData(BaseRuntimeData runtimeData)
    {
        DataManager.Instance.SaveRuntimeData(runtimeData);
    }

    protected List<BaseStaticDataLoader> GetStaticData(BaseStage stage, int dataindex)
    {
        return DataManager.Instance.GetStaticData(stage.StageKind, dataindex);
    }

    protected List<BaseRuntimeData> GetRuntimeData(BaseStage stage, int dataindex)
    {
        return DataManager.Instance.GetRuntimeData(stage.StageKind, dataindex);
    }
}
```
</details>

---

### 3. 커스텀 필드 기반 확장 가능한 공용 인벤토리 시스템

#### 개요
5개 게임이 각자 다른 아이템 속성을 가지면서도 **하나의 공통 인벤토리 시스템**을 사용할 수 있도록 설계한 시스템입니다. `IItemable` 인터페이스로 아이템의 공통 계약을 정의하고, `Dictionary<string, string>` 기반의 커스텀 필드로 게임별 고유 속성을 확장합니다.

#### 시스템 구조
```
IItemStaticData (게임별 시트 데이터)
    ↓ ToItemData()
LUPItemData (공통 아이템 데이터)
  ├─ 필수 필드: ItemID, ItemName, ItemType, MaxStackSize, Icon, ...
  └─ 커스텀 필드: Dictionary<string, string> (게임별 확장)
    ↓ ItemManager.LoadAllItems()
Dictionary<int, LUPItemData> (전역 아이템 DB)
    ↓
Inventory (BaseRuntimeData 상속)
  ├─ Dictionary<string, InventorySlot> (런타임 슬롯 관리)
  ├─ ISerializationCallbackReceiver (JSON 직렬화 호환)
  └─ 이벤트: OnItemAdded, OnItemRemoved, OnItemUsed
    ↓
InventoryManager (게임별 인벤토리 키 기반 관리)
```

#### 기술적 특징
- **IItemable 인터페이스** : 아이템의 공통 계약(ID, 이름, 타입, 스택, 아이콘, 사용 가능 여부)을 정의하면서, `GetInt()`, `GetFloat()`, `GetString()`, `GetBool()` 메서드로 게임별 커스텀 속성에 타입 안전하게 접근합니다.
- **ISerializationCallbackReceiver** : Unity의 `JsonUtility`는 `Dictionary`를 직렬화하지 못하므로, `OnBeforeSerialize`에서 `Dictionary<string, InventorySlot>` → `List<InventorySlotData>`로 변환하고, `InitializeFromJson`에서 다시 복원하는 구조로 JSON 호환성을 확보했습니다.
- **ItemManager 리플렉션 로딩** : 여러 게임의 `BaseStaticDataLoader`에서 `DataList` 필드를 리플렉션으로 접근하여, `IItemStaticData`를 구현한 데이터만 선별적으로 `LUPItemData`로 변환합니다. 동일 ID 아이템은 `MergeWith()`로 커스텀 필드를 병합합니다.
- **스택 관리** : 동일 아이템이 이미 존재하면 스택 수량을 증가시키고, 최대 스택 초과 시 새 슬롯을 자동 생성합니다. 슬롯 키는 `{ItemID}_{counter}` 형태로 고유성을 보장합니다.
- **이벤트 기반 알림** : `OnItemAdded`, `OnItemRemoved`, `OnItemUsed` 이벤트로 UI나 퀘스트 시스템이 인벤토리 변경을 구독할 수 있습니다.

<details>
<summary><b>🔍 코드 — IItemable & LUPItemData 커스텀 필드</b></summary>

```csharp
public interface IItemable
{
    int ItemID { get; }
    string ItemName { get; }
    LUP.Define.ItemType Type { get; }
    int MaxStackSize { get; }
    Sprite Icon { get; }
    string Description { get; }
    bool IsUsable { get; }
    void OnUse();

    // 커스텀 필드 접근 — 게임별 고유 속성을 타입 안전하게 조회
    int GetInt(string fieldName, int defaultValue = 0);
    float GetFloat(string fieldName, float defaultValue = 0f);
    string GetString(string fieldName, string defaultValue = "");
    bool GetBool(string fieldName, bool defaultValue = false);
    bool HasCustomField(string fieldName);
}

[System.Serializable]
public class LUPItemData : IItemable
{
    [Header("필수 필드")]
    [SerializeField] private int itemID;
    [SerializeField] private string itemName;
    [SerializeField] private Define.ItemType itemType;
    [SerializeField] private int maxStackSize = 1;

    // 게임별 확장 필드 — Dictionary로 자유롭게 추가
    [System.NonSerialized]
    private Dictionary<string, string> customFields = new Dictionary<string, string>();

    // 타입 안전 접근자
    public int GetInt(string fieldName, int defaultValue = 0)
    {
        if (customFields.TryGetValue(fieldName, out string value))
            return int.TryParse(value, out int result) ? result : defaultValue;
        return defaultValue;
    }

    // 다른 로더에서 온 동일 ID 아이템의 커스텀 필드 병합
    public void MergeWith(LUPItemData other)
    {
        if (other?.customFields == null) return;
        foreach (var kvp in other.customFields)
        {
            if (!customFields.ContainsKey(kvp.Key) || string.IsNullOrEmpty(customFields[kvp.Key]))
                customFields[kvp.Key] = kvp.Value;
        }
    }
}
```
</details>

<details>
<summary><b>🔍 코드 — Inventory JSON 직렬화 (ISerializationCallbackReceiver)</b></summary>

```csharp
[Serializable]
public class Inventory : BaseRuntimeData, ISerializationCallbackReceiver
{
    // JSON 직렬화용 — JsonUtility가 Dictionary를 지원하지 않으므로 List로 변환
    [SerializeField]
    private List<InventorySlotData> _serializedSlots = new List<InventorySlotData>();

    // 런타임 사용 — O(1) 조회를 위한 Dictionary
    [System.NonSerialized]
    private Dictionary<string, InventorySlot> slots;

    public event Action<IItemable, int> OnItemAdded;
    public event Action<IItemable, int> OnItemRemoved;
    public event Action<IItemable> OnItemUsed;

    // 직렬화 직전: Dictionary → List 변환
    public void OnBeforeSerialize()
    {
        _serializedSlots.Clear();
        if (slots == null) return;
        foreach (var kvp in slots)
        {
            _serializedSlots.Add(new InventorySlotData
            {
                slotKey = kvp.Key,
                itemID = kvp.Value.Item.ItemID,
                quantity = kvp.Value.Quantity
            });
        }
    }

    // JSON 로드 후: List → Dictionary 복원 (ItemManager에서 아이템 참조 재연결)
    public void InitializeFromJson()
    {
        slots = new Dictionary<string, InventorySlot>();
        foreach (var slotData in _serializedSlots)
        {
            IItemable item = ItemManager.Instance.GetItem(slotData.itemID);
            if (item != null)
                slots[slotData.slotKey] = new InventorySlot(item, slotData.quantity);
        }
    }

    public bool AddItem(IItemable item, int quantity = 1)
    {
        // 1단계: 스택 가능한 기존 슬롯 탐색
        InventorySlot stackableSlot = null;
        foreach (var slot in slots.Values)
        {
            if (slot.Item.ItemID == item.ItemID && slot.CanStack)
            { stackableSlot = slot; break; }
        }

        if (stackableSlot != null)
        {
            if (stackableSlot.TryAddQuantity(quantity))
            {
                OnItemAdded?.Invoke(item, quantity);
                NotifyValueChanged(); // → BaseRuntimeData 자동 저장 트리거
                return true;
            }
        }

        // 2단계: 새 슬롯 생성
        string slotKey = GenerateSlotKey(item.ItemID);
        slots[slotKey] = new InventorySlot(item, quantity);
        OnItemAdded?.Invoke(item, quantity);
        NotifyValueChanged();
        return true;
    }
}
```
</details>

---

### 4. Pull Request 관리

#### 개요
13명의 프로그래머가 동시에 작업하는 환경에서 코드 품질을 유지하고 충돌을 최소화하기 위해 Fork 기반의 PR 워크플로우를 운영했습니다.

#### 수행 내용
- **코드 리뷰** : 모든 PR에 대해 코드 리뷰를 수행하여 프레임워크 규약 준수 여부와 코드 품질을 검증했습니다.
- **머지 전략** : 각 게임팀의 작업이 서로 충돌하지 않도록 브랜치 전략을 수립하고, 충돌 발생 시 해결을 지원했습니다.
- **시스템 사용 가이드** : 프레임워크 시스템(데이터 연동, 인벤토리, Stage 등)의 사용법을 문서화하여 팀원들에게 제공했습니다.

- [Project LUP 프레임워크 사용 가이드 (Notion)](https://www.notion.so/Project-LUP-299894b38b10818bac41e103ee63d3a9)
- [인벤토리 시스템 문서 (Notion)](https://www.notion.so/LUP-2c0894b38b108039a1fbf7eaafc7e50d)

## 🔧 트러블슈팅
### JsonUtility의 Dictionary 미지원으로 인한 인벤토리 직렬화 문제
인벤토리 시스템을 저장/로드하는 과정에서, `Dictionary<string, InventorySlot>`이 `JsonUtility.ToJson()`에서 빈 객체로 직렬화되는 문제를 발견했습니다.
원인을 분석한 결과, Unity의 `JsonUtility`는 `Dictionary` 타입을 직렬화하지 않는다는 것을 확인했습니다.
이를 해결하기 위해 `ISerializationCallbackReceiver` 인터페이스를 구현하여, Dictionary를 List로 변환하고, 로드 시 `ItemManager`를 통해 아이템 참조를 재연결하며 Dictionary를 복원하는 2단계 직렬화 구조를 설계했습니다.
이 과정에서 "처음부터 List만 사용하면 되지 않았을까?"라는 의문이 들었으나, 런타임에서 아이템 조회/추가/삭제의 빈도가 높아 `O(1)` 접근이 가능한 Dictionary의 사용을 유지하기로 결정했습니다.
대신, 직렬화 시에만 List로 변환하여 `JsonUtility` 호환성과 런타임 성능을 모두 확보하는 구조를 채택했습니다.

### 5개 게임의 아이템 속성 차이로 인한 인벤토리 확장성 문제
공용 인벤토리 시스템을 설계하던 중, 각 게임의 아이템이 서로 다른 고유 속성을 가지고 있어 하나의 아이템 클래스로 통합할 수 없는 문제에 직면했습니다.
처음에는 게임별 아이템 클래스를 각각 만드는 방안을 고려했으나, 이 경우 인벤토리 시스템이 게임별 타입에 종속되어 공용성을 잃게 됩니다.
이를 해결하기 위해 **필수 필드 + 커스텀 필드 분리 구조**를 도입했습니다. `LUPItemData`에 모든 게임이 공통으로 사용하는 필드(ID, 이름, 타입, 스택)를 정의하고, 게임별 고유 속성은 `Dictionary<string, string>` 형태의 커스텀 필드에 저장합니다.
또한, `ICustomFieldSupport` 인터페이스를 통해 Google Sheets에서 `[Column]`으로 매핑되지 않은 나머지 컬럼들이 자동으로 커스텀 필드에 수집되도록 하여, **시트에 새 컬럼을 추가해도 프레임워크 코드 수정 없이 데이터가 확장**되는 구조를 완성했습니다.

### 에디터 환경에서의 코루틴 실행 불가 문제
데이터 연동 시스템을 구현하던 중, 에디터의 커스텀 인스펙터에서 "데이터 읽기" 버튼을 클릭했을 때 `UnityWebRequest` 기반 코루틴이 실행되지 않는 문제를 발견했습니다.
원인을 분석한 결과, `StartCoroutine`은 `MonoBehaviour`의 `Update` 루프에 의존하는데, 에디터 인스펙터 컨텍스트에서는 게임 루프가 동작하지 않아 코루틴이 진행되지 않는 것을 확인했습니다.
이를 해결하기 위해 `async/await` 패턴으로 `IEnumerator`를 래핑하는 `ProcessCoroutine` 메서드를 구현했습니다. 이 메서드는 `MoveNext()`로 코루틴을 수동 진행하면서, `UnityWebRequestAsyncOperation`이 반환될 경우 `isDone`을 폴링하는 방식으로 비동기 웹 요청을 처리합니다. 중첩된 코루틴도 재귀적으로 처리하여 어떤 깊이의 코루틴도 에디터에서 정상 동작하도록 했습니다.
