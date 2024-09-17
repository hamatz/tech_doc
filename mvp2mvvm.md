# MVPからMVVMへの段階的移行ガイド

## 1. はじめに

大規模なAndroidアプリケーションをMVP（Model-View-Presenter）アーキテクチャからMVVM（Model-View-ViewModel）アーキテクチャに移行する過程は、複雑で時間のかかるプロセスです。さらに、MVVMに移行した後も、巨大なViewModelの最適化が必要になることがあります。このガイドでは、MVPからMVVMへの段階的な移行方法と、その後のViewModel最適化について説明します。

## 2. 移行プロセスの概要

移行プロセスは以下の主要なステップで構成されます：

1. CommonInterfaceの導入
2. 巨大Presenterの段階的分割
3. 共通UIコンポーネントの管理
4. PresenterからViewModelへの段階的移行
5. Jetpack Composeの導入
6. 巨大ViewModelの最適化

## 3. 詳細なプロセス

### 3.1 CommonInterfaceの導入

最初のステップとして、ViewとPresenter/ViewModelの間にCommonInterfaceを導入します。

```kotlin
interface CommonInterface {
    fun loadData()
    fun processUserAction(action: String)
    // 他の共通メソッド
}
```

このインターフェースを導入することで、後続の変更をより容易に行うことができます。

#### CommonInterfaceの役割と限界

CommonInterfaceは以下の点で有用です：

- 共通の操作を定義し、ViewがPresenterやViewModelと相互作用する際の一貫性を提供します。
- 段階的な移行を可能にし、一部の画面がPresenterを使用し、他の画面がViewModelを使用する中間状態を管理しやすくします。

しかし、以下の限界があります：

- PresenterとViewModelの根本的な違い（例：ライフサイクル管理、状態保持など）をすべて抽象化することは困難です。
- ViewModelの特定の機能（例：LiveData、StateFlow、SavedStateHandle）をインターフェースで完全にカバーすることは難しいです。

### 3.2 巨大Presenterの段階的分割

巨大なPresenterを一度に分割するのではなく、段階的に小さな部分に分けていきます。

1. 初期状態（巨大Presenter）
2. 一部機能の切り出し
3. さらなる機能の切り出し
4. 最終的な分割状態

各段階で、以下のステップを繰り返します：

a. 特定の機能を特定し、新しいPresenterとして切り出す  
b. 元のPresenterから切り出した機能を削除する  
c. 新しいPresenterとCommonInterfaceを接続する  
d. テストを追加/更新し、機能が正しく動作することを確認する  

```kotlin
class PresenterA : CommonInterface {
    override fun loadData() {
        // 実装
    }
    // 他のメソッド
}

class RemainingPresenter : CommonInterface {
    // 残りの機能
}
```

### 3.3 共通UIコンポーネントの管理

複数のタブや画面で共通して使用されるUIコンポーネント（例：通知ダイアログ、Spinner、ヘッダー、フッターなど）の管理は、アプリケーションのアーキテクチャに大きな影響を与えます。これらのコンポーネントを適切に管理することで、巨大Presenterの問題を軽減し、MVVMへの移行をスムーズに行うことができます。

#### a. 共通UIコンポーネント用のViewModel作成

共通UIコンポーネント専用のViewModelを作成します。このViewModelは、アプリケーション全体で共有されるUIの状態を管理します。

```kotlin
class SharedUIViewModel : ViewModel() {
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    private val _notificationMessage = MutableStateFlow<String?>(null)
    val notificationMessage: StateFlow<String?> = _notificationMessage.asStateFlow()

    fun showLoading() {
        _isLoading.value = true
    }

    fun hideLoading() {
        _isLoading.value = false
    }

    fun showNotification(message: String) {
        _notificationMessage.value = message
    }

    fun clearNotification() {
        _notificationMessage.value = null
    }
}
```

#### b. 依存性注入の活用

Dagger HiltやKoinなどの依存性注入フレームワークを使用して、SharedUIViewModelをアプリケーション全体で共有します。

```kotlin
@HiltViewModel
class SharedUIViewModel @Inject constructor() : ViewModel() {
    // 実装
}

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    private val sharedUIViewModel: SharedUIViewModel by viewModels()

    // 実装
}
```

#### c. 共通UIコンポーネントの表示ロジック

共通UIコンポーネントの表示ロジックを、各画面のViewModelから分離し、ActivityやFragmentレベルで管理します。

```kotlin
class MainActivity : AppCompatActivity() {
    private val sharedUIViewModel: SharedUIViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        lifecycleScope.launch {
            sharedUIViewModel.isLoading.collect { isLoading ->
                showHideLoadingSpinner(isLoading)
            }
        }

        lifecycleScope.launch {
            sharedUIViewModel.notificationMessage.collect { message ->
                message?.let { showNotificationDialog(it) }
            }
        }
    }

    private fun showHideLoadingSpinner(show: Boolean) {
        // スピナーの表示/非表示ロジック
    }

    private fun showNotificationDialog(message: String) {
        // 通知ダイアログの表示ロジック
    }
}
```

#### d. 各タブ/画面のViewModelからの利用

各タブや画面のViewModelは、必要に応じてSharedUIViewModelを参照し、共通UIコンポーネントの状態を更新します。

```kotlin
@HiltViewModel
class TabViewModel @Inject constructor(
    private val sharedUIViewModel: SharedUIViewModel
) : ViewModel() {

    fun loadData() {
        sharedUIViewModel.showLoading()
        viewModelScope.launch {
            try {
                // データ読み込みロジック
                sharedUIViewModel.hideLoading()
            } catch (e: Exception) {
                sharedUIViewModel.hideLoading()
                sharedUIViewModel.showNotification("エラーが発生しました")
            }
        }
    }
}
```

この方法により、共通UIコンポーネントの管理を中央集権化し、各タブや画面のViewModelをよりスリムに保つことができます。

### 3.4 PresenterからViewModelへの段階的移行

Presenterを十分に小さく分割したら、各PresenterをViewModelに移行し始めます。この過程は完全に透過的ではない可能性があるため、以下の実践的なアプローチを採用します：

#### a. 段階的な移行：

1. CommonInterfaceを実装したシンプルなViewModelから始めます。
2. 徐々にViewModel特有の機能を追加していきます。

```kotlin
// 初期段階のViewModel（CommonInterfaceを実装）
class UserViewModel : ViewModel(), CommonInterface {
    override fun loadData() {
        // ViewModelの実装（初期段階ではPresenterと似た実装）
    }
    // 他のメソッド
}

// 段階的に進化したViewModel
class EvolutionaryUserViewModel : ViewModel(), CommonInterface {
    private val _userData = MutableStateFlow<UserData?>(null)
    val userData: StateFlow<UserData?> = _userData.asStateFlow()

    override fun loadData() {
        viewModelScope.launch {
            // 非同期でデータを取得
            _userData.value = fetchUserData()
        }
    }
    // 他のメソッド
}
```

#### b. アダプターパターンの使用：

PresenterとViewModelの間にアダプターレイヤーを導入します。

```kotlin
class UserViewModelAdapter(private val viewModel: UserViewModel) : CommonInterface {
    override fun loadData() {
        viewModel.loadData()
    }
    // 他のメソッド
}
```

#### c. View側の段階的な更新：

1. 初期段階では、ViewはCommonInterfaceを通じて画面描画します。
2. ViewModel特有の機能（LiveData、StateFlowなど）を利用する準備ができたら、View側のコードを更新します。

```kotlin
// 初期段階
class UserProfileActivity : AppCompatActivity() {
    private lateinit var userDataProvider: CommonInterface

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        userDataProvider = ViewModelProvider(this).get(UserViewModel::class.java)
        userDataProvider.loadData()
        // UIの更新ロジック（従来の方法）
    }
}

// 進化後
class EvolutionaryUserProfileActivity : AppCompatActivity() {
    private val viewModel: EvolutionaryUserViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            viewModel.userData.collect { userData ->
                // UIの更新ロジック（Flow使用）
            }
        }
        viewModel.loadData()
    }
}
```

### 3.5 Jetpack Composeの導入

従来のViewをJetpack Composeに段階的に移行します。

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val uiState by viewModel.uiState.collectAsState()
    
    // Compose UI
}
```

### 3.6 巨大ViewModelの最適化

MVVMに移行した後、時間の経過とともに一部のViewModelが肥大化し、管理が難しくなる場合があります。このような巨大ViewModelを最適化するためのアプローチを以下に示します。

#### a. 責任ごとのViewModel分割

巨大なViewModelを、異なる責任や機能ドメインに基づいて複数の小さなViewModelに分割します。

例：
```kotlin
// 分割前の巨大ViewModel
class UserProfileViewModel : ViewModel() {
    // ユーザー情報、投稿リスト、フォロワーリストなどの全ての状態と操作
}

// 分割後
class UserInfoViewModel : ViewModel() {
    // ユーザーの基本情報に関する状態と操作
}

class UserPostsViewModel : ViewModel() {
    // ユーザーの投稿リストに関する状態と操作
}

class UserFollowersViewModel : ViewModel() {
    // ユーザーのフォロワーリストに関する状態と操作
}
```

#### b. UseCase/Interactorの導入

ビジネスロジックをViewModelから分離し、UseCase（またはInteractor）に移動させます。これにより、ViewModelはUI状態の管理に集中できます。

```kotlin
class GetUserInfoUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(userId: String): UserInfo {
        return userRepository.getUserInfo(userId)
    }
}

class UserInfoViewModel @Inject constructor(
    private val getUserInfoUseCase: GetUserInfoUseCase
) : ViewModel() {
    private val _userInfo = MutableStateFlow<UserInfo?>(null)
    val userInfo: StateFlow<UserInfo?> = _userInfo.asStateFlow()

    fun loadUserInfo(userId: String) {
        viewModelScope.launch {
            _userInfo.value = getUserInfoUseCase(userId)
        }
    }
}
```

#### c. 状態管理の分離

UI状態の管理を専用のStateHolderクラスに委譲します。これにより、ViewModelの役割がさらに明確になり、テストも容易になります。

```kotlin
data class UserInfoState(
    val userInfo: UserInfo? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

class UserInfoViewModel @Inject constructor(
    private val getUserInfoUseCase: GetUserInfoUseCase
) : ViewModel() {
    private val _state = MutableStateFlow(UserInfoState())
    val state: StateFlow<UserInfoState> = _state.asStateFlow()

    fun loadUserInfo(userId: String) {
        viewModelScope.launch {
            _state.value = _state.value.copy(isLoading = true)
            try {
                val userInfo = getUserInfoUseCase(userId)
                _state.value = _state.value.copy(userInfo = userInfo, isLoading = false)
            } catch (e: Exception) {
                _state.value = _state.value.copy(error = e.message, isLoading = false)
            }
        }
    }
}
```

#### d. 共有ViewModelの活用

複数の画面やフラグメントで共有される状態がある場合、ActivityやNavigation Graphのスコープで共有ViewModelを使用します。

```kotlin
@HiltViewModel
class SharedUserViewModel @Inject constructor() : ViewModel() {
    private val _userId = MutableStateFlow<String?>(null)
    val userId: StateFlow<String?> = _userId.asStateFlow()

    fun setUserId(id: String) {
        _userId.value = id
    }
}

// 使用例（Fragment）
@AndroidEntryPoint
class UserProfileFragment : Fragment() {
    private val sharedViewModel: SharedUserViewModel by activityViewModels()
    private val userInfoViewModel: UserInfoViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.lifecycleScope.launch {
            sharedViewModel.userId.collect { userId ->
                userId?.let { userInfoViewModel.loadUserInfo(it) }
            }
        }
    }
}
```

## 4. 移行プロセスの図解

以下は、全体的な移行プロセスを視覚化した図です。

```mermaid
graph TD

    subgraph "6. ViewModel最適化"
        A6[Screen 1] --> B6[ViewModel A1]
        A6 --> E6[ViewModel A2]
        Z6[Screen 2] --> F6[ViewModel B]
        Y6[Screen 3] --> H6[ViewModel C]
        B6 --> I6[UseCase]
        E6 --> I6
        F6 --> I6
        H6 --> I6
        I6 --> C6[Model]
        G6[SharedUIViewModel] --> A6
        G6 --> Z6
        G6 --> Y6
    end

    subgraph "5. 完全MVVM構造（Compose導入）"
        A5[Screen 1] --> B5[ViewModel A]
        Z5[Screen 2] --> E5[ViewModel B]
        Y5[Screen 3] --> F5[ViewModel C]
        B5 --> C5[Model]
        E5 --> C5
        F5 --> C5
        G5[SharedUIViewModel] --> A5
        G5 --> Z5
        G5 --> Y5
    end

    subgraph "4. PresenterとViewModel混在"
        A4[View 1] --> B4[ViewModel A]
        Z4[View 2] --> E4[ViewModel B]
        Y4[View 3] --> F4[Presenter C]
        B4 --> C4[Model]
        E4 --> C4
        F4 --> C4
        G4[SharedUIViewModel] --> A4
        G4 --> Z4
        G4 --> Y4
    end

    subgraph "3. 共通UIコンポーネント管理の導入"
        A3[View 1] --> D3[CommonInterface]
        Z3[View 2] --> D3
        Y3[View 3] --> D3
        D3 --> B3[Presenter A]
        D3 --> E3[Presenter B]
        D3 --> F3[Presenter C]
        B3 --> C3[Model]
        E3 --> C3
        F3 --> C3
        G3[SharedUIViewModel] --> A3
        G3 --> Z3
        G3 --> Y3
    end

    subgraph "2. Presenter分割とCommonInterface導入"
        A2[View 1] --> D2[CommonInterface]
        Z2[View 2] --> D2
        Y2[View 3] --> D2
        D2 --> B2[Presenter A]
        D2 --> E2[Presenter B]
        D2 --> F2[Presenter C]
        B2 --> C2[Model]
        E2 --> C2
        F2 --> C2
    end

    subgraph "1. 初期MVP構造（巨大Presenter）"
        A1[View 1] --> B1[巨大Presenter]
        Z1[View 2] --> B1
        Y1[View 3] --> B1
        B1 --> C1[Model]
    end

    1 -->|"Presenter分割とCommonInterface導入"| 2
    2 -->|"共通UIコンポーネント管理の導入"| 3
    3 -->|"段階的なViewModel移行"| 4
    4 -->|"完全なMVVM化とCompose導入"| 5
    5 -->|"ViewModel最適化"| 6

    classDef default fill:#f9f,stroke:#333,stroke-width:2px;
    classDef interface fill:#ff9,stroke:#333,stroke-width:2px;
    classDef viewmodel fill:#9f9,stroke:#333,stroke-width:2px;
    classDef compose fill:#f99,stroke:#333,stroke-width:2px;
    classDef bigPresenter fill:#f66,stroke:#333,stroke-width:2px;
    classDef sharedViewModel fill:#9ff,stroke:#333,stroke-width:2px;
    classDef useCase fill:#ff9,stroke:#333,stroke-width:2px;

    class D2,D3 interface;
    class B4,E4,B5,E5,F5,B6,E6,F6,H6 viewmodel;
    class A5,Z5,Y5,A6,Z6,Y6 compose;
    class B1 bigPresenter;
    class G3,G4,G5,G6 sharedViewModel;
    class I6 useCase;
```

## 5. 実装時の注意点

1. リファクタリングの優先順位を決定する：最も問題のある、または変更頻度の高い部分から始める
2. 各段階でテストを追加・更新し、既存機能が破壊されていないことを確認する
3. パフォーマンスを継続的にモニタリングする
4. チーム全体で設計の一貫性を維持する
5. 段階的な変更を行い、各段階で動作確認とコードレビューを行う
6. CommonInterfaceの限界を理解し、必要に応じてアダプターパターンを使用する
7. View側のコードも段階的に更新し、ViewModel特有の機能を活用できるようにする
8. 共通UIコンポーネントの状態管理を中央集権化し、各ViewModelから適切に利用する
9. 依存性注入を活用して、SharedUIViewModelを効率的に共有する
10. ViewModelの責任を明確に定義し、必要に応じて分割する
11. ビジネスロジックをUseCaseに抽出し、ViewModelをUI状態管理に集中させる
12. 状態管理のためのStateHolderクラスの使用を検討する
13. 共有が必要な状態については、適切なスコープの共有ViewModelを使用する

## 6. 結論

MVPからMVVMへの移行、そしてその後のViewModel最適化は、継続的な改善プロセスの一部です。このガイドで説明した段階的なアプローチを採用することで、リスクを最小限に抑えつつ、徐々にアプリケーションアーキテクチャをモダンなものとし、最適化することができます。

初期の移行段階では、CommonInterfaceやアダプターパターンなどのテクニックが有用です。その後、ViewModelが肥大化した場合は、責任の分割、UseCaseの導入、状態管理の分離などの手法を用いて最適化を行います。

共通UIコンポーネントの管理や共有ViewModelの適切な使用は、アプリケーション全体の一貫性と効率性を高めます。また、Jetpack Composeの導入により、UI層の柔軟性と表現力が向上します。

このプロセス全体を通じて、コードの保守性、テスト容易性、そして拡張性が向上し、長期的にはアプリケーションの品質と開発効率の向上につながります。各段階で学んだことを活かし、必要に応じてプロセスを調整しながら進めることが重要です。

最終的には、責任が明確で、テスト可能で保守性の高い、モダンなアーキテクチャを持つアプリケーションを実現することができるでしょう。この継続的な改善プロセスは、アプリケーションの寿命を通じて続けられるべきものです。
