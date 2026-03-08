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

> 5개 게임(로그라이크, 덱 전략, 생산/건설/강화, 익스트랙션 슈터, 슈팅)이 하나의 프로젝트 안에서 공통 프레임워크를 공유하는 구조

**[프로젝트 저장소](https://github.com/project-lup/projectlup-v2025)** | **[프레임워크 가이드 (Notion)](https://www.notion.so/Project-LUP-299894b38b10818bac41e103ee63d3a9)** | **[인벤토리 문서 (Notion)](https://www.notion.so/LUP-2c0894b38b108039a1fbf7eaafc7e50d)**

---

## 담당 시스템
1. [Google Sheets 데이터 연동 시스템](#1-google-sheets-데이터-연동-시스템)
2. [Stage 전환 관리 시스템](#2-stage-전환-관리-시스템)
3. [공용 인벤토리 시스템](#3-공용-인벤토리-시스템)

---

## 1. Google Sheets 데이터 연동 시스템

### Adapter 패턴 - 데이터 소스 추상화

<details>
<summary><b>IDataSourceAdapter</b></summary>

```csharp
public interface IDataSourceAdapter
{
    IEnumerator LoadData(string url, System.Action<string> onSuccess, System.Action<string> onError);
    List<T> ParseToObjects<T>(string rawData, int startRow) where T : new();
}
```
</details>

<details>
<summary><b>CSVDataSourceAdapter - 리플렉션 기반 자동 매핑</b></summary>

```csharp
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

    public List<T> ParseToObjects<T>(string csvData, int startRow) where T : new()
    {
        string[] lines = CSVParser.SplitLines(csvData);
        int headerIndex = startRow - 1;

        string[] headers = CSVParser.ParseLine(lines[headerIndex]);
        Dictionary<string, int> headerMap = new Dictionary<string, int>();
        for (int i = 0; i < headers.Length; i++)
            headerMap[headers[i].Trim()] = i;

        List<T> dataList = new List<T>();

        for (int i = headerIndex + 1; i < lines.Length; i++)
        {
            if (string.IsNullOrWhiteSpace(lines[i])) continue;
            string[] values = CSVParser.ParseLine(lines[i]);
            T data = ParseDataRow<T>(values, headerMap);
            if (data != null) dataList.Add(data);
        }
        return dataList;
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

        // [Column]으로 매핑되지 않은 나머지 컬럼 → 커스텀 필드에 자동 수집
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

    private void SetFieldValue(FieldInfo field, object instance, string value)
    {
        if (field.FieldType.IsEnum)
        {
            if (System.Enum.TryParse(field.FieldType, value, true, out object enumValue))
                field.SetValue(instance, enumValue);
        }
        else if (field.FieldType == typeof(string)) field.SetValue(instance, value);
        else if (field.FieldType == typeof(int) && int.TryParse(value, out int intVal))
            field.SetValue(instance, intVal);
        else if (field.FieldType == typeof(float) && float.TryParse(value, out float floatVal))
            field.SetValue(instance, floatVal);
        else if (field.FieldType == typeof(bool) && bool.TryParse(value, out bool boolVal))
            field.SetValue(instance, boolVal);
    }
}
```
</details>

### ColumnAttribute

```csharp
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
```

### BaseStaticDataLoader - 어댑터 팩토리 + ScriptableObject 직렬화

<details>
<summary><b>BaseStaticDataLoader</b></summary>

```csharp
public abstract class BaseStaticDataLoader : ScriptableObject
{
    [SerializeField] public Define.DataSourceType sourceType = Define.DataSourceType.CSV;
    [SerializeField] public string associatedWorksheet = "";
    [SerializeField] public int START_ROW = 1;

    public abstract IEnumerator LoadSheet();
}

public abstract class BaseStaticDataLoader<T> : BaseStaticDataLoader where T : new()
{
    [SerializeField] public List<T> DataList = new List<T>();
    protected abstract string CSV_URL { get; }

    public override IEnumerator LoadSheet()
    {
        IDataSourceAdapter adapter = CreateAdapter(sourceType);
        string url = GetURL(sourceType);

        string rawData = null;
        string error = null;

        yield return adapter.LoadData(url,
            data => rawData = data,
            err => error = err);

        if (!string.IsNullOrEmpty(error)) yield break;

        // 어댑터가 리플렉션으로 CSV → List<T> 자동 변환
        DataList = adapter.ParseToObjects<T>(rawData, START_ROW);
    }

    private IDataSourceAdapter CreateAdapter(Define.DataSourceType type)
    {
        switch (type)
        {
            case Define.DataSourceType.CSV: return new CSVDataSourceAdapter();
            default: throw new System.NotImplementedException($"Adapter for {type} not implemented");
        }
    }
}
```
</details>

### 에디터 커스텀 인스펙터 - async/await로 코루틴 래핑

<details>
<summary><b>BaseStaticDataReaderEditor</b></summary>

```csharp
#if UNITY_EDITOR
[CustomEditor(typeof(BaseStaticDataLoader), true)]
public class BaseStaticDataReaderEditor : Editor
{
    BaseStaticDataLoader data;
    void OnEnable() { data = (BaseStaticDataLoader)target; }

    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        GUILayout.Label("\n\n스프레드 시트 읽어오기");
        if (GUILayout.Button("데이터 읽기"))
            LoadDataAsync();
    }

    private async void LoadDataAsync()
    {
        IEnumerator coroutine = data.LoadSheet();
        await ProcessCoroutine(coroutine);
        EditorUtility.SetDirty(data);
        AssetDatabase.SaveAssets();
    }

    // 에디터에서는 MonoBehaviour.StartCoroutine 사용 불가
    // → async/await로 IEnumerator를 수동 진행
    private async System.Threading.Tasks.Task ProcessCoroutine(IEnumerator coroutine)
    {
        while (coroutine.MoveNext())
        {
            object current = coroutine.Current;
            if (current == null)
                await System.Threading.Tasks.Task.Delay(10);
            else if (current is IEnumerator nestedCoroutine)
                await ProcessCoroutine(nestedCoroutine);  // 중첩 코루틴 재귀 처리
            else if (current is UnityWebRequestAsyncOperation asyncOp)
            {
                while (!asyncOp.isDone)
                    await System.Threading.Tasks.Task.Delay(100);
            }
            else
                await System.Threading.Tasks.Task.Delay(10);
        }
    }
}
#endif
```
</details>

---

## 2. Stage 전환 관리 시스템

<details>
<summary><b>StageManager - 전환 테이블 검증 + 코루틴 파이프라인</b></summary>

```csharp
public class StageManager : Singleton<StageManager>
{
    private Dictionary<Define.StageKind, SceneList> sceneNameMap = new Dictionary<Define.StageKind, SceneList>();
    private List<List<Define.StageKind>> transitionTable = new List<List<Define.StageKind>>();
    private bool isTransitioning = false;

    public void LoadStage(Define.StageKind targetStageKind, int sceneindex = -1)
    {
        if (isTransitioning) return;
        if (!IsValidTransition(currentStageKind, targetStageKind)) return;
        StartCoroutine(TransitionCoroutine(targetStageKind, sceneindex));
    }

    private bool IsValidTransition(Define.StageKind from, Define.StageKind to)
    {
        return transitionTable[(int)from].Contains(to);
    }

    private IEnumerator TransitionCoroutine(Define.StageKind targetStageKind, int sceneindex = -1)
    {
        isTransitioning = true;

        // Stage Exit (SaveDatas + FadeOut)
        yield return StartCoroutine(OnStageExit());

        // 씬 이름 조회
        string sceneName = (sceneindex == -1)
            ? sceneNameMap[targetStageKind].sceneNames[0]
            : sceneNameMap[targetStageKind].sceneNames[sceneindex];

        // 비동기 씬 로드
        AsyncOperation asyncLoad = SceneManager.LoadSceneAsync(sceneName);
        while (!asyncLoad.isDone)
            yield return new WaitForSeconds(0.1f);

        // 새 Stage 인스턴스 탐색
        currentStageInstance = FindFirstObjectByType<BaseStage>();
        while (currentStageInstance == null)
        {
            currentStageInstance = FindFirstObjectByType<BaseStage>();
            yield return new WaitForSeconds(0.1f);
        }

        // Stage Enter (LoadResources + GetDatas + FadeIn)
        yield return StartCoroutine(OnStageEnter());
        currentStageKind = targetStageKind;
        isTransitioning = false;
    }

    private IEnumerator FadeOut()
    {
        fadeCanvas.blocksRaycasts = true;
        float elapsed = 0f;
        while (elapsed < fadeDuration)
        {
            elapsed += Time.deltaTime;
            fadeCanvas.alpha = Mathf.Lerp(0f, 1f, elapsed / fadeDuration);
            yield return null;
        }
        fadeCanvas.alpha = 1f;
    }

    private IEnumerator FadeIn()
    {
        float elapsed = 0f;
        while (elapsed < fadeDuration)
        {
            elapsed += Time.deltaTime;
            fadeCanvas.alpha = Mathf.Lerp(1f, 0f, elapsed / fadeDuration);
            yield return null;
        }
        fadeCanvas.alpha = 0f;
        fadeCanvas.blocksRaycasts = false;
    }
}
```
</details>

<details>
<summary><b>BaseStage - Template Method 패턴 생명주기</b></summary>

```csharp
public abstract class BaseStage : MonoBehaviour
{
    [ReadOnly] public Define.StageKind StageKind = Define.StageKind.Main;

    // Template Method — 프레임워크가 호출 순서를 제어
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

    public virtual IEnumerator OnStageExit()
    {
        SaveDatas();
        yield return null;
    }

    // 각 게임 팀이 구현하는 추상 메서드
    protected abstract void LoadResources();
    protected abstract void GetDatas();
    protected abstract void SaveDatas();

    // DataManager 연동 유틸리티
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

## 3. 공용 인벤토리 시스템

### LUPItemData - 필수 필드 + 커스텀 필드

<details>
<summary><b>LUPItemData</b></summary>

```csharp
[System.Serializable]
public class LUPItemData : IItemable
{
    [Header("필수 필드")]
    [SerializeField] private int itemID;
    [SerializeField] private string itemName;
    [SerializeField] private Define.ItemType itemType;
    [SerializeField] private string iconPath = "";
    [SerializeField] private int maxStackSize = 1;
    [SerializeField] private bool isUsable = false;
    [SerializeField] private string description = "";

    [System.NonSerialized] private Sprite _iconCache;

    // 게임별 확장 필드 — Dictionary로 자유롭게 추가
    [System.NonSerialized]
    private Dictionary<string, string> customFields = new Dictionary<string, string>();
    [SerializeField]
    private List<CustomField> serializedCustomFields = new List<CustomField>();

    public Sprite Icon
    {
        get
        {
            if (_iconCache == null && !string.IsNullOrEmpty(iconPath))
                _iconCache = Resources.Load<Sprite>(iconPath);
            return _iconCache;
        }
    }

    // 타입 안전 접근자
    public int GetInt(string fieldName, int defaultValue = 0)
    {
        if (customFields.TryGetValue(fieldName, out string value))
            return int.TryParse(value, out int result) ? result : defaultValue;
        return defaultValue;
    }

    public float GetFloat(string fieldName, float defaultValue = 0f)
    {
        if (customFields.TryGetValue(fieldName, out string value))
            return float.TryParse(value, out float result) ? result : defaultValue;
        return defaultValue;
    }

    public string GetString(string fieldName, string defaultValue = "")
    {
        return customFields.TryGetValue(fieldName, out string value) ? value : defaultValue;
    }

    public bool GetBool(string fieldName, bool defaultValue = false)
    {
        if (customFields.TryGetValue(fieldName, out string value))
            return bool.TryParse(value, out bool result) ? result : defaultValue;
        return defaultValue;
    }

    // 동일 ID 아이템의 커스텀 필드 병합
    public void MergeWith(LUPItemData other)
    {
        if (other?.customFields == null) return;
        if (customFields == null) customFields = new Dictionary<string, string>();

        foreach (var kvp in other.customFields)
        {
            if (!customFields.ContainsKey(kvp.Key) || string.IsNullOrEmpty(customFields[kvp.Key]))
                customFields[kvp.Key] = kvp.Value;
        }
        SyncToSerializedList();
    }
}
```
</details>

### ItemManager - 리플렉션 기반 멀티 게임 아이템 로딩

<details>
<summary><b>ItemManager::LoadAllItems</b></summary>

```csharp
public class ItemManager : Singleton<ItemManager>
{
    [SerializeField] private List<BaseStaticDataLoader> loaders = new List<BaseStaticDataLoader>();
    private Dictionary<int, LUPItemData> itemDatabase = new Dictionary<int, LUPItemData>();

    public IEnumerator LoadAllItems()
    {
        itemDatabase.Clear();

        foreach (var loader in loaders)
        {
            if (loader == null) continue;

            // 리플렉션으로 DataList 필드 접근
            var dataListField = loader.GetType().GetField("DataList");
            if (dataListField == null) continue;

            var dataList = dataListField.GetValue(loader) as System.Collections.IList;
            if (dataList == null || dataList.Count == 0) continue;

            foreach (var staticData in dataList)
            {
                // IItemStaticData 인터페이스 구현 여부 확인
                if (staticData is IItemStaticData itemStaticData)
                {
                    var itemData = itemStaticData.ToItemData();

                    // 같은 ID 아이템이 이미 있으면 커스텀 필드 병합
                    if (itemDatabase.ContainsKey(itemData.ItemID))
                        itemDatabase[itemData.ItemID].MergeWith(itemData);
                    else
                        itemDatabase[itemData.ItemID] = itemData;
                }
            }
            yield return null;  // 프레임 분산
        }
    }

    public IItemable GetItem(int itemID)
    {
        return itemDatabase.TryGetValue(itemID, out LUPItemData item) ? item : null;
    }
}
```
</details>

### Inventory - ISerializationCallbackReceiver로 Dictionary ↔ List 변환

<details>
<summary><b>Inventory</b></summary>

```csharp
[Serializable]
public class Inventory : BaseRuntimeData, ISerializationCallbackReceiver
{
    // JSON 직렬화용 — JsonUtility가 Dictionary 미지원
    [SerializeField]
    private List<InventorySlotData> _serializedSlots = new List<InventorySlotData>();

    // 런타임 사용 — O(1) 조회
    [System.NonSerialized]
    private Dictionary<string, InventorySlot> slots;

    public event Action<IItemable, int> OnItemAdded;
    public event Action<IItemable, int> OnItemRemoved;
    public event Action<IItemable> OnItemUsed;

    // 직렬화 직전: Dictionary → List
    public void OnBeforeSerialize()
    {
        SyncToSerializedList();
    }

    private void SyncToSerializedList()
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

    // JSON 로드 후: List → Dictionary 복원 (ItemManager에서 참조 재연결)
    public void InitializeFromJson()
    {
        slots = new Dictionary<string, InventorySlot>();

        foreach (var slotData in _serializedSlots)
        {
            if (slotData == null || slotData.itemID == 0) continue;

            IItemable item = ItemManager.Instance.GetItem(slotData.itemID);
            if (item != null)
                slots[slotData.slotKey] = new InventorySlot(item, slotData.quantity);
        }
    }
}
```
</details>

### InventoryManager - 키 기반 인벤토리 관리

<details>
<summary><b>InventoryManager</b></summary>

```csharp
public class InventoryManager : Singleton<InventoryManager>
{
    private Dictionary<string, Inventory> inventories = new Dictionary<string, Inventory>();

    public void RegisterInventory(string inventoryKey, Inventory inventory)
    {
        inventories[inventoryKey] = inventory;
    }

    public Inventory GetInventory(string inventoryKey)
    {
        return inventories.TryGetValue(inventoryKey, out Inventory inventory) ? inventory : null;
    }

    public Inventory LoadOrCreateInventory(string inventoryKey, string filename)
    {
        Inventory inventory;

        if (JsonDataHelper.FileExists(filename))
        {
            inventory = JsonDataHelper.LoadData<Inventory>(filename);
            if (inventory != null)
            {
                inventory.filename = filename;
                inventory.InitializeFromJson();  // Dictionary 복원
            }
            else
            {
                inventory = new Inventory();
                inventory.filename = filename;
            }
        }
        else
        {
            inventory = new Inventory();
            inventory.filename = filename;
        }

        RegisterInventory(inventoryKey, inventory);
        return inventory;
    }

    public void SaveAllInventories()
    {
        foreach (var kvp in inventories)
        {
            if (kvp.Value != null) kvp.Value.SaveData();
        }
    }
}
```
</details>
