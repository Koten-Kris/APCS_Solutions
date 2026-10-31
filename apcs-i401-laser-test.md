# [解題報告] APCS i401 雷射測試

* 題目來源：[ZeroJudge i401 題目連結](https://zerojudge.tw/ShowProblem?problemid=i401)
## 核心資料結構設計：如何避免 TLE？
這題的資料量高達 25 萬，如果用暴力法逐格搜尋絕對會 TLE。為此，我們需要利用二分搜尋來快速查找下一面鏡子。

為了達到高速查找，我宣告了兩個主要的雜湊表：
* `X`: Key 為 X 座標，Value 儲存該直排上所有鏡子的 Y 座標。
* `Y`: Key 為 Y 座標，Value 儲存該橫排上所有鏡子的 X 座標。

> **為什麼選擇 unordered_map / defaultdict？**
> 本題也可以使用標準的 map，但 map 底層是紅黑樹，查詢時間複雜度為 O(log N)；而 unordered_map 底層是雜湊表（Hash Table），平均查詢時間為 O(1)。
> 雖然雜湊表在極少數極端狀況下可能因為「雜湊碰撞（Hash Collision）」導致效能退化到 O(N)，但在這題中並不會發生這樣的問題，所以可以放心使用！
> *注意：為了能在陣列中使用二分搜尋，讀入所有座標後，必須先對每個 Key 對應的內容進行排序（Sort），時間複雜度為 O(N log N)。*

>**記憶體效能補充**：這題其實也可以直接開一個長度為 600001 的陣列（我有測試過了），雖然能獲得最快的 O(1) 隨機存取速度，但因為題目測資屬於「稀疏資料（Sparse Data）」，直接開大陣列會產生大量未使用的空值，導致嚴重的**記憶體空間浪費**。*
> *因此，使用雜湊字典雖然在查找時多了一些雜湊運算的時間開銷，但能「有用到再配置記憶體」，在空間利用率上遠勝過直接開大陣列！*

---

## 狀態與變數定義
為了精確模擬雷射光的軌跡，我定義了以下控制變數：
1. `bool J`: 代表搜尋方向。`true` / `True` 往座標大的地方找（右/上），`false` / `False` 往座標小的地方找（左/下）。
2. `pair<char,int> CS`(Current State): 記錄目前雷射是在 X 軸（直排）還是 Y 軸（橫排）上移動，以及目前的 Key 值。
3. `int CP`(Current Place): 記錄目前所在的橫排或直排的位置座標。
   * *註：因為雷射從 (0, 0) 向右發射，前兩次移動的 CP 初始化必須為 0，之後再依據找到的鏡子動態更新。*
4. `Total = 0`: 記錄雷射反射的總次數。

---

## 反射與轉向邏輯
每一次模擬，都使用二分搜尋法在對應的陣列中尋找下一面鏡子。如果找不到，代表雷射射出邊界，程式終止。

當成功找到下一面鏡子時：
* **反射次數累加**：`Total += 1`
* **方向切換（關鍵發現）**：經過觀察，方向指標 `J`（往大或往小找）只有在鏡子狀態為 `1`（也就是 / 鏡子）時需要改變。若原本往大找，就改為往小找，反之亦然。而另一種鏡子狀態則不會改變 `J` 的搜尋方向。
* **位置更新**：將 `CP` 更新為這面新鏡子的座標，並切換 `CS` 的動態軸向（直排變橫排，橫排變直排）。
## Python的轉換
1. `map`改成`defaultdick`，雖然用`dick`也可，但是這樣子在加入新Key時，就會需要再多打一行判斷式，為了程式碼的美觀，所以我一律使用`defaultdict`!
2. `lower_bound`和upper_bound`一律換成`bisect_left`和`bisect_right`
>*注意:`bisect`和`bound`回傳的性質差異*
# 程式碼
# C++
```CPP
#include <bits/stdc++.h>
// 競賽常用巨集：快速簡化 sort 與二分搜尋的範圍代碼
#define all(A) (A).begin(),(A).end()
using namespace std;

// 宣告全域雜湊表：
// X: Key 為 X 座標，Value 為該直排上所有鏡子的 {Y座標, 鏡子種類}
// Y: Key 為 Y 座標，Value 為該橫排上所有鏡子的 {X座標, 鏡子種類}
unordered_map<int, vector<pair<int, int>>> X, Y;

int main()
{
    // I/O 輸入輸出優化，打斷 cin/cout 綁定，避免大量資料時 TLE
    ios_base::sync_with_stdio(false);
    cin.tie(0);
    
    int T, C, R, S, Total = 0, V, CP = 0;
    cin >> T;
    
    //讀入所有鏡子的資料
    for (V = 0; V < T; V++)
    {
        cin >> C >> R >> S;
        X[C].push_back({R, S});
        Y[R].push_back({C, S});
    }
    
    //關鍵優化：利用 all(A) 巨集對每個直排與橫排的鏡子進行排序，以利後續二分搜尋
    for (auto& B : X) sort(all(B.second));
    for (auto& B : Y) sort(all(B.second));
    
    // 檢查起始橫排(Y)是否有任何鏡子，若有才開始模擬雷射軌跡
    if (!Y[0].empty())
    {
        bool J = true;                 //方向指標：true 往大找（右/上），false 往小找（左/下）
        pair<char, int> CS = {'Y', 0}; //Current State: {目前軸向 'X' 或 'Y', 目前的 Key 值}
        
        while (true)
        {
            //三元運算子：根據目前軸向（X 或 Y）動態取得對應的鏡子 vector 
            auto& N = (CS.first == 'Y') ? Y[CS.second] : X[CS.second];
            
            if (J)
            {
                //─── 情況 A：往座標「大」的地方搜尋 ───
                //用 upper_bound 找嚴格大於當前位置 CP 的下一個鏡子
                //傳入 make_pair(CP, 2) 確保能精準越過當前座標，找到下一個，因為鏡子的狀態都是0或1，所以用2可以保證絕對的嚴格大於
                auto NP = upper_bound(all(N), make_pair(CP, 2));
                
                if (NP != N.end()) //成功在邊界內找到下一面鏡子
                {
                    //特殊邏輯：前兩次移動時 CP 必須為 0（因為從原點出發），之後更新為上一次的 Key
                    if (Total > 0) CP = CS.second;
                    
                    CS.first = (CS.first == 'Y') ? 'X' : 'Y'; //軸向切換
                    CS.second = NP->first;                   //更新 Key 值為新鏡子的座標
                    
                    //轉向規律：若碰到狀態為 1 的鏡子（\），搜尋方向 J 反轉
                    if (NP->second == 1) J = false;
                }
                else break; //跑出邊界，結束模擬
            }
            else
            {
                //─── 情況 B：往座標「小」的地方搜尋 ───
                //用 lower_bound 尋找大於或等於當前位置 CP 的第一個位置
                auto NP = lower_bound(all(N), make_pair(CP, 0));
                
                if (NP != N.begin()) //確保左邊還有鏡子可以退（不超出左邊界）
                {
                    NP--; //往左挪一格，即為小於當前 CP 且最鄰近的鏡子
                    
                    if (Total > 0) CP = CS.second;
                    
                    CS.first = (CS.first == 'Y') ? 'X' : 'Y';
                    CS.second = NP->first;
                    
                    //轉向規律：若往小找時碰到狀態為 1 的鏡子（\），搜尋方向 J 反轉回 true
                    if (NP->second == 1) J = true;
                }
                else break; //跑出邊界，結束模擬
            }
            Total++; //反射次數加 1
        }
    }
    cout << Total; //輸出最終總反射次數
}
```
# Python
```Python
import sys
from collections import defaultdict
import bisect as idx

#使用 sys.stdin.readline 進行高速輸入優化，避免 I/O 導致 TLE
input = sys.stdin.readline

#讀取鏡子總數
T = int(input())

#使用 defaultdict 初始化雜湊表，自動將未生成的 Key 對應到空的 list
X, Y = defaultdict(list), defaultdict(list)

#讀入所有鏡子的座標與狀態
for L in range(T):
    C, R, S = map(int, input().split())
    X[C].append([R, S])  # 直排 X 紀錄：[Y座標, 鏡子種類]
    Y[R].append([C, S])  # 橫排 Y 紀錄：[X座標, 鏡子種類]
    
#關鍵優化：對每個直排與橫排的鏡子依座標大小進行排序，以利後續二分搜尋
for L in X:
    X[L].sort(key=lambda X: X[0])
for L in Y:
    Y[L].sort(key=lambda X: X[0])
    
Total = 0  # 紀錄雷射反射總次數

#檢查起始橫排(Y)是否有任何鏡子，若有才開始模擬
if Y[0]:
    CP = 0             #Current Place: 紀錄目前雷射在當前軸向的座標位置
    CS = ['Y', 0]      #Current State: ['目前軸向', 當前的 Key 值]
    J = True           #方向指標: True 代表往座標大的地方找，False 代表往小的地方找
    
    while True:
        #根據目前的軸向（X 或 Y）動態切換搜尋的雜湊表
        NP = Y[CS[1]] if CS[0] == 'Y' else X[CS[1]]
        
        if J:
            #─── 情況 A：往座標「大」的地方搜尋 ───
            #使用 bisect_right 尋找嚴格大於當前位置 CP 的下一個鏡子索引
            N = idx.bisect_right(NP, CP, key=lambda X: X[0])
            
            if N < len(NP):  #成功找到下一面鏡子
                #特殊邏輯：前兩次移動時 CP 必須為 0（因為從原點出發），之後才更新為上一次的 Key
                CP = CS[1] if Total > 0 else 0
                
                #軸向切換：橫排變直排，直排變橫排
                CS[0] = 'X' if CS[0] == 'Y' else 'Y'
                CS[1] = NP[N][0]  # 更新 Key 值為新鏡子的座標
                
                #轉向規律：若碰到狀態為 1 的鏡子（\），搜尋方向 J 反轉
                if NP[N][1] == 1:
                    J = False
            else:
                break  #跑出地圖邊界，結束模擬
                
        else:
            #─── 情況 B：往座標「小」的地方搜尋 ───
            #使用 bisect_left 尋找大於或等於當前位置 CP 的索引
            N = idx.bisect_left(NP, CP, key=lambda X: X[0])
            
            #安全檢查，確保位置與目前的 CP 不同後，往左挪一格（找到更小的鄰近鏡子）
            if NP[N] != CP:
                N -= 1
                CP = CS[1] if Total> 0 else 0
                CS[0] = 'X' if CS[0] == 'Y' else 'Y'
                CS[1] = NP[N][0]
                
                #轉向規律：若往小找時碰到狀態為 1 的鏡子（\），搜尋方向 J 反轉回 True
                if NP[N][1] == 1:
                    J = True
            else:
                break  #跑出邊界或找不到更小的鏡子，結束模擬
                
        Total += 1  #反射次數加 1
        
print(Total)
```
<small>*後記：這是我寫的第一篇正式解題報告。當初在寫這題時，在 ZeroJudge 上吃了很多次的 TLE。後來發現只要資料量一稀疏，大陣列就會浪費極多記憶體，最後想通了改用 unordered_map 搭配二分搜尋法優化，才終於順利 AC！真的非常有成就感。*</small>
