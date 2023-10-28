# Unity_CyberPlantAR
<img src="https://capsule-render.vercel.app/api?type=waving&color=auto&height=200&section=header&text=CyberPlant-AR&fontSize=80" /> 

# AR 기반 EDG Local Control Panel 기동 절차 시스템

## Features (담당 기능)

-   [사용자 로그인 및 신규 등록 기능, 점검 기록지 구현](#panel-procedure-manager)
-   [Graph Viewer 구현](#graphicsLine-setting)
    
## Panel Procedure Manager

>사용된 스크립트<br/>
> ProcedureList.cs

시나리오 절차를 구성하고 세팅, 관리하는 기능을 구현하였습니다.

```c#

public class ProcedureList : MonoBehaviour
{
    public static ProcedureList instance;

    #region PROPERTIES
    DataTable DT;

    Dictionary<int, string> procedureRowDic;
    List<GameObject> procedureList;
    public GameObject prevList;
    int currentPage = 1;
    int totalPage = 0;

    [Header("List Props")]
    public GameObject subListPrefab;
    public Button listOnButton;
    public Button listOffButton;
    public Transform listPanel;
    public GameObject listPrefab;
    public GameObject pagePrefab;
    public Transform listLayout;
    public Text decriptionText;
    public Button prevPageButton;
    public Button nextPageButton;
    public Text pageText;
    public Text guideText;

    [Header("Color Props")]
    public float popupDuration;
    public Sprite highlightSprite;
    public Sprite normalSprite;

    public Color highlightTextColor;
    public Color normalTextColor;
    public Color highlightBoxColor;
    public Color normalBoxColor;
    public int rowCount = 6;

    public GameObject TableName;
    #endregion


    void Awake()
    {
        
            instance = this;

        SetPopupEffect();
        SetPageFunction();
    }
    void OnDestroy()
    {
        instance = null;
    }

    void SetPopupEffect()
    {
        listOnButton.onClick.AddListener(
            delegate
            {
                listPanel.gameObject.SetActive(!listPanel.gameObject.activeSelf);
            });
    }
    
    //절차(Procedure) 중복제거 저장
    void LoadData()
    {
        procedureRowDic = new Dictionary<int, string>();

        string procedureText = "0";
        string temp = "";
        for (int i = 0; i < ProjectManager.instance.seqTable.Rows.Count; i++)
        {
            temp = ProjectManager.instance.seqTable.Rows[i]["Procedure"].ToString();
            if (string.IsNullOrEmpty(temp)) continue;

            if (!procedureText.Equals(temp))
            {
                procedureText = temp;
                procedureRowDic.Add(i, procedureText);
            }
        }
    }
    //초기 설정들 (데이터 로드 및 리스트 생성)
    public void Set()
    {
        decriptionText.transform.parent.gameObject.SetActive(true);
        LoadData();
        Create();
        
        for (int i = 0; i < ProjectManager.instance.seqTable.Rows.Count; i++)
        {
            int rowIndex = 0;
            if (!string.IsNullOrEmpty(ProjectManager.instance.seqTable.Rows[i]["SubList"].ToString()))
            {
                rowIndex = i;
                GameObject subListObj = Instantiate(subListPrefab, ProjectManager.instance.selectSubSeq.transform);
                subListObj.GetComponentInChildren<Button>().onClick.AddListener(delegate
                {
                    SkipTo(rowIndex);
                });
                subListObj.GetComponentInChildren<Text>().text = ProjectManager.instance.seqTable.Rows[i]["Guide"].ToString();
            }
        }
        
    }

    //현재 진행중인 절차 표시 및 절차가 있는 페이지로 자동 넘겨줌
    public void ShowCurrent(DataRow dataRow)
    {
        //현재 절차 지시사항(Description) 띄우기
        if (decriptionText)
            decriptionText.text = dataRow["Description"].ToString();

        //이전리스트 원래상태로 되돌리기
        if (prevList != null)
        {
            prevList.transform.GetChild(0).GetComponent<Image>().sprite = normalSprite;
            prevList.transform.GetChild(0).GetComponent<Image>().color = normalBoxColor;
            prevList.GetComponent<RectTransform>().localScale = Vector3.one;
            prevList.GetComponentsInChildren<Text>()[0].color = normalTextColor;
            prevList.GetComponentsInChildren<Text>()[1].color = normalTextColor;
        }
        //현재 리스트 표시
        GameObject currentList = procedureList.Find(x => x.GetComponentsInChildren<Text>()[1].text.Equals(dataRow["Procedure"].ToString()));
        currentList.transform.GetChild(0).GetComponent<Image>().sprite = highlightSprite;
        currentList.transform.GetChild(0).GetComponent<Image>().color = highlightBoxColor;
        currentList.GetComponent<RectTransform>().localScale = Vector3.one * 1.03f;
        currentList.GetComponentsInChildren<Text>()[0].color = highlightTextColor;
        currentList.GetComponentsInChildren<Text>()[1].color = highlightTextColor;
        prevList = currentList;

        //페이지 자동찾기
        string page = currentList.transform.parent.name.Split('_')[1];
        currentPage = int.Parse(page);
        pageText.text = currentPage + " / " + totalPage;
        listLayout.transform.Find("Page_" + currentPage).SetAsFirstSibling();
    }

    //절차(Procedure) 리스트 생성
    void Create()
    {
        //Fast & Slow 텍스트 출력 
        string ScenarioName = ProjectManager.instance.tableName;
        string[] resultSc =  ScenarioName.Split('_');
        TableName.GetComponent<Text>().text = resultSc[0].ToString();

        resultSc = null;

        procedureList = new List<GameObject>();

        //설정한 row Count에 따른 페이지 수 계산
        if (procedureRowDic.Count % rowCount != 0)
            totalPage = (int)(procedureRowDic.Count / rowCount) + 1;
        else if (procedureRowDic.Count % rowCount == 0)
            totalPage = procedureRowDic.Count / rowCount;
        else if (procedureRowDic.Count % rowCount < 1)
            totalPage = 1;

        //페이지 생성
        int index = 0;
        for (int i = 0; i < totalPage; i++)
        {
            GameObject page = Instantiate(pagePrefab, listLayout);
            page.name = "Page_" + (i + 1);
            //리스트 생성
            for (int j = 0; j < rowCount; j++)
            {
                //생성한 procedure에 해당하는 열번호 저장
                int rowIndex = procedureRowDic.ElementAt(index).Key;

                if (! ProjectManager.instance.seqTable.Rows[rowIndex]["SubList"].ToString().Equals("TRUE"))
                {
                    GameObject listObj = Instantiate(listPrefab, page.transform);
                    listObj.GetComponentInChildren<Button>().onClick.AddListener(delegate
                    {
                        //스킵기능
                        SkipTo(rowIndex);
                    });
                    //리스트 텍스트 및 색상 초기 설정
                    listObj.GetComponentsInChildren<Text>()[0].text = (index + 1).ToString("000") + ". ";
                    listObj.GetComponentsInChildren<Text>()[1].text = procedureRowDic.ElementAt(index).Value;
                    listObj.GetComponentsInChildren<Text>()[2].text = "[ " + ProjectManager.instance.seqTable.Rows[rowIndex]["Location"].ToString() + " ]";
                    listObj.GetComponentsInChildren<Text>()[0].color = normalTextColor;
                    listObj.GetComponentsInChildren<Text>()[1].color = normalTextColor;

                    procedureList.Add(listObj);
                    index++;
                    if (index == procedureRowDic.Count) break;
                }
            }
        }
    }

    //페이지 버튼 이벤트 설정
    public void SetPageFunction()
    {
        //이전페이지로
        prevPageButton.onClick.AddListener(
            delegate
            {
                //if (currentPage == 1) return; ==> 페이지 순환형식 막고싶을때 주석해제
                if (currentPage == 1)
                    currentPage = totalPage;
                else currentPage -= 1;

                pageText.text = currentPage + " / " + totalPage;
                listLayout.transform.Find("Page_" + currentPage).SetAsFirstSibling();
            });
        //다음페이지로
        nextPageButton.onClick.AddListener(
            delegate
            {
                //if (currentPage == totalPage) return; ==> 페이지 순환형식 막고싶을때 주석해제
                if (currentPage == totalPage)
                    currentPage = 1;
                else currentPage += 1;

                pageText.text = currentPage + " / " + totalPage;
                listLayout.transform.Find("Page_" + currentPage).SetAsFirstSibling();
            });
    }

    //절차(Procedure) 간 점프
    public void SkipTo(int rowIndex)
    {
        ProjectManager.instance.StopCoroutine(ProjectManager.instance.mainCor); // 진행중인 메인코루틴 중지
        ProjectManager.instance.rowIndex = rowIndex; //선택한 절차의 열번호 설정
        ProjectManager.instance.Next_Btn(); //초기화 및 다음 시퀀스 준비
        ProjectManager.instance.mainCor = ProjectManager.instance.StartCoroutine(ProjectManager.instance.MainProcess()); //메인코루틴 시작
    }

    //행(rowIndex) 간 점프기능 (Description 점프)
    public void OnClick_DescriptionSkip_Btn(GameObject btnObj)
    {
        //첫번째 시퀀스 혹은 마지막 시퀀스가 아닐 때.
        int resultRowIndex = 0;

        if (btnObj.name.Contains("Prev") && ProjectManager.instance.rowIndex != 0)
            resultRowIndex = ProjectManager.instance.rowIndex - 1;
        else if (btnObj.name.Contains("Next") && ProjectManager.instance.rowIndex < ProjectManager.instance.seqTable.Rows.Count - 1)
            resultRowIndex = ProjectManager.instance.rowIndex + 1;

        SkipTo(resultRowIndex);
    }

    //데이터 및 리스트 생성 이전으로 초기화
    public void Initialize()
    {
        currentPage = 1;
        prevList = null;

        decriptionText.transform.parent.gameObject.SetActive(false);

        listPanel.gameObject.SetActive(false);
        //생성한 리스트 전부 삭제
        for (int i = 0; i < listLayout.childCount; i++)
            Destroy(listLayout.GetChild(i).gameObject);
    }
}
```

## GraphicsLine Setting

>사용된 스크립트<br/>
> GraphicsLineRenderer.cs 

데이터를 기반으로 Line Graph Viewer 기능을 구현하였습니다.
```c#

[RequireComponent(typeof(MeshRenderer))]
[RequireComponent(typeof(MeshFilter))]
public class GraphicsLineRenderer : MonoBehaviour
{
    public Material lmat;
    private Mesh ml;
    private Vector3 s;

    private float lineSize = 0.1f;
    private bool firstQuad = true;

    public void SetMesh(Mesh ml)
    {
        GetComponent<MeshFilter>().mesh = ml;
    }

    public void SetMaterial(Material lmat)
    {
        GetComponent<MeshRenderer>().material = lmat;
    }

    public void setWidth(float width)
    {
        lineSize = width;
    }

    public void AddPoint(Vector3 point)
    {
        if (s != Vector3.zero)
        {
            AddLine(GetComponent<MeshFilter>().mesh, MakeQuad(s, point, lineSize, firstQuad));
            firstQuad = false;
        }

        s = point;
    }

    Vector3[] MakeQuad(Vector3 s, Vector3 e, float w, bool all)
    {
        w = w / 2;

        Vector3[] q;
        if (all)
        {
            q = new Vector3[4];
        }
        else
        {
            q = new Vector3[2];
        }

        Vector3 n = Vector3.Cross(s, e);
        Vector3 l = Vector3.Cross(n, e - s);
        l.Normalize();

        if (all)
        {
            q[0] = transform.InverseTransformPoint(s + l * w);
            q[1] = transform.InverseTransformPoint(s + l * -w);
            q[2] = transform.InverseTransformPoint(e + l * w);
            q[3] = transform.InverseTransformPoint(e + l * -w);
        }
        else
        {
            q[0] = transform.InverseTransformPoint(s + l * w);
            q[1] = transform.InverseTransformPoint(s + l * -w);
        }
        return q;
    }

    void AddLine(Mesh m, Vector3[] quad)
    {
        int vl = m.vertices.Length;

        Vector3[] vs = m.vertices;
        vs = resizeVertices(vs, 2 * quad.Length);

        for (int i = 0; i < 2 * quad.Length; i += 2)
        {
            vs[vl + i] = quad[i / 2];
            vs[vl + i + 1] = quad[i / 2];
        }

        Vector2[] uvs = m.uv;
        uvs = resizeUVs(uvs, 2 * quad.Length);

        if (quad.Length == 4)
        {
            uvs[vl] = Vector2.zero;
            uvs[vl + 1] = Vector2.zero;
            uvs[vl + 2] = Vector2.right;
            uvs[vl + 3] = Vector2.right;
            uvs[vl + 4] = Vector2.up;
            uvs[vl + 5] = Vector2.up;
            uvs[vl + 6] = Vector2.one;
            uvs[vl + 7] = Vector2.one;
        }
        else
        {
            if (vl % 8 == 0)
            {
                uvs[vl] = Vector2.zero;
                uvs[vl + 1] = Vector2.zero;
                uvs[vl + 2] = Vector2.right;
                uvs[vl + 3] = Vector2.right;

            }
            else
            {
                uvs[vl] = Vector2.up;
                uvs[vl + 1] = Vector2.up;
                uvs[vl + 2] = Vector2.one;
                uvs[vl + 3] = Vector2.one;
            }
        }

        int tl = m.triangles.Length;

        int[] ts = m.triangles;
        ts = resizeTriangles(ts, 12);

        if (quad.Length == 2)
        {
            vl -= 4;
        }

        // front-facing quad
        ts[tl] = vl;
        ts[tl + 1] = vl + 2;
        ts[tl + 2] = vl + 4;

        ts[tl + 3] = vl + 2;
        ts[tl + 4] = vl + 6;
        ts[tl + 5] = vl + 4;

        // back-facing quad
        ts[tl + 6] = vl + 5;
        ts[tl + 7] = vl + 3;
        ts[tl + 8] = vl + 1;

        ts[tl + 9] = vl + 5;
        ts[tl + 10] = vl + 7;
        ts[tl + 11] = vl + 3;

        m.vertices = vs;
        m.uv = uvs;
        m.triangles = ts;
        m.RecalculateBounds();
        m.RecalculateNormals();
    }

    Vector3[] resizeVertices(Vector3[] ovs, int ns)
    {
        Vector3[] nvs = new Vector3[ovs.Length + ns];
        for (int i = 0; i < ovs.Length; i++)
        {
            nvs[i] = ovs[i];
        }

        return nvs;
    }

    Vector2[] resizeUVs(Vector2[] uvs, int ns)
    {
        Vector2[] nvs = new Vector2[uvs.Length + ns];
        for (int i = 0; i < uvs.Length; i++)
        {
            nvs[i] = uvs[i];
        }

        return nvs;
    }

    int[] resizeTriangles(int[] ovs, int ns)
    {
        int[] nvs = new int[ovs.Length + ns];
        for (int i = 0; i < ovs.Length; i++)
        {
            nvs[i] = ovs[i];
        }

        return nvs;
    }
}

```

<img src="https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4bce9982-9a99-459c-8698-46a1d81eedb1/Untitled.png" width="400" height="250"/>
