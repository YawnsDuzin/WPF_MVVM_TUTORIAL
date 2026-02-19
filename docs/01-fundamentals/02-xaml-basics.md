# XAML 기본 문법

> 이 문서는 WinForms의 Designer.cs에 익숙한 개발자를 대상으로,
> WPF의 UI 정의 언어인 XAML을 처음부터 설명합니다.

---

## 목차

1. [XAML이란?](#1-xaml이란)
2. [WinForms의 Designer.cs와 XAML 비교](#2-winforms의-designercs와-xaml-비교)
3. [XAML 기본 구조와 네임스페이스](#3-xaml-기본-구조와-네임스페이스)
4. [자주 쓰는 컨트롤과 예제](#4-자주-쓰는-컨트롤과-예제)
5. [레이아웃 시스템](#5-레이아웃-시스템)
6. [리소스와 스타일](#6-리소스와-스타일)
7. [WinForms 개발자가 헷갈리는 점들](#7-winforms-개발자가-헷갈리는-점들)

---

## 1. XAML이란?

**XAML(eXtensible Application Markup Language, "재믈"로 발음)** 은
XML 기반의 마크업 언어로, WPF에서 UI를 선언적으로 정의하는 데 사용됩니다.

### 핵심 개념

- XAML은 **C# 객체를 XML 형태로 표현**한 것입니다.
- XAML의 모든 태그는 결국 **C# 클래스의 인스턴스**입니다.
- XAML의 속성(attribute)은 **C# 클래스의 프로퍼티(property)** 에 대응됩니다.

### XAML = C# 객체 생성의 선언적 표현

```xml
<!-- XAML로 버튼 생성 -->
<Button Content="클릭하세요" Width="100" Height="30" Background="LightBlue" />
```

위 XAML은 아래 C# 코드와 **완전히 동일**합니다:

```csharp
// C# 코드로 버튼 생성 (XAML과 동일한 결과)
var button = new Button();
button.Content = "클릭하세요";
button.Width = 100;
button.Height = 30;
button.Background = new SolidColorBrush(Colors.LightBlue);
```

### Python 개발자를 위한 비유

```python
# Python Tkinter: 코드로 UI 구성
button = tk.Button(root, text="클릭하세요", width=10, bg="lightblue")
button.pack()
```

```xml
<!-- WPF XAML: 마크업으로 UI 구성 (같은 결과) -->
<Button Content="클릭하세요" Width="100" Background="LightBlue" />
```

---

## 2. WinForms의 Designer.cs와 XAML 비교

WinForms에서 폼 디자이너를 사용하면 `Form1.Designer.cs`에
자동 생성 코드가 만들어졌습니다. XAML은 이 역할을 대체합니다.

### WinForms의 Designer.cs

```csharp
// Form1.Designer.cs — WinForms 자동 생성 코드 (읽기 어려움)
private void InitializeComponent()
{
    this.label1 = new System.Windows.Forms.Label();
    this.textBox1 = new System.Windows.Forms.TextBox();
    this.button1 = new System.Windows.Forms.Button();
    this.SuspendLayout();

    // label1
    this.label1.AutoSize = true;
    this.label1.Location = new System.Drawing.Point(12, 15);
    this.label1.Name = "label1";
    this.label1.Size = new System.Drawing.Size(37, 15);
    this.label1.Text = "이름:";

    // textBox1
    this.textBox1.Location = new System.Drawing.Point(55, 12);
    this.textBox1.Name = "textBox1";
    this.textBox1.Size = new System.Drawing.Size(200, 23);

    // button1
    this.button1.Location = new System.Drawing.Point(12, 50);
    this.button1.Name = "button1";
    this.button1.Size = new System.Drawing.Size(243, 30);
    this.button1.Text = "확인";
    this.button1.Click += new EventHandler(this.button1_Click);

    // Form1
    this.Controls.Add(this.label1);
    this.Controls.Add(this.textBox1);
    this.Controls.Add(this.button1);
    this.ClientSize = new System.Drawing.Size(280, 100);
    this.Text = "폼 타이틀";
    this.ResumeLayout(false);
}
```

### 같은 UI를 XAML로 작성

```xml
<!-- MainWindow.xaml — 사람이 읽기 쉬운 마크업 -->
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="폼 타이틀" Width="300" Height="140">

    <StackPanel Margin="12">
        <!-- 이름 입력 영역 -->
        <StackPanel Orientation="Horizontal" Margin="0,0,0,10">
            <TextBlock Text="이름:" VerticalAlignment="Center" Margin="0,0,10,0" />
            <TextBox x:Name="textBox1" Width="200" />
        </StackPanel>

        <!-- 확인 버튼 -->
        <Button Content="확인" Height="30" Click="Button1_Click" />
    </StackPanel>
</Window>
```

### 비교 요약

| 특성 | WinForms Designer.cs | WPF XAML |
|---|---|---|
| **형식** | C# 코드 (자동 생성) | XML 기반 마크업 |
| **가독성** | 낮음 (좌표, 크기 숫자 나열) | 높음 (계층 구조가 명확) |
| **편집** | 디자이너 전용 (수동 수정 비권장) | 직접 편집 가능 & 권장 |
| **레이아웃** | 절대 좌표 (Location 속성) | 상대적 레이아웃 (패널 시스템) |
| **실수 시** | 디자이너가 깨질 수 있음 | XML 문법 오류만 나면 됨 |

---

## 3. XAML 기본 구조와 네임스페이스

### XAML 파일의 기본 구조

```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:MyApp"
        mc:Ignorable="d"
        Title="MainWindow" Height="450" Width="800">

    <!-- 여기에 UI 요소를 배치 -->
    <Grid>

    </Grid>
</Window>
```

### 네임스페이스(xmlns) 설명

```xml
<!-- ① 기본 WPF 네임스페이스: WPF의 모든 기본 컨트롤이 여기에 있음 -->
<!-- Button, TextBox, Grid, StackPanel 등을 접두사 없이 사용 가능 -->
xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"

<!-- ② XAML 언어 네임스페이스: XAML 자체의 기능 제공 -->
<!-- x:Class, x:Name, x:Key, x:Type 등에서 사용 -->
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"

<!-- ③ 디자인 타임 네임스페이스: Visual Studio 디자이너 전용 -->
<!-- d:DesignHeight, d:DesignWidth 등 디자이너에서만 적용되는 속성 -->
xmlns:d="http://schemas.microsoft.com/expression/blend/2008"

<!-- ④ 마크업 호환성: 디자인 타임 속성을 런타임에 무시하도록 설정 -->
xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"

<!-- ⑤ 로컬 네임스페이스: 현재 프로젝트의 C# 클래스를 XAML에서 사용 -->
<!-- 예: <local:MyCustomControl /> -->
xmlns:local="clr-namespace:MyApp"
```

### 자주 사용하는 xmlns 접두사 관례

```xml
<!-- 다른 프로젝트 또는 어셈블리의 클래스를 참조할 때 -->
xmlns:models="clr-namespace:MyApp.Models"
xmlns:vm="clr-namespace:MyApp.ViewModels"
xmlns:converters="clr-namespace:MyApp.Converters"

<!-- NuGet 패키지의 네임스페이스 참조 -->
xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes"
```

### x:Class, x:Name, x:Key

```xml
<!-- x:Class: 이 XAML이 연결되는 C# 클래스 (코드 비하인드) -->
<Window x:Class="MyApp.MainWindow">
    <!--
        → MyApp 네임스페이스의 MainWindow 클래스와 연결됨
        → MainWindow.xaml.cs의 partial class MainWindow 와 합쳐짐
    -->

    <!-- x:Name: 컨트롤에 이름을 부여 (코드 비하인드에서 참조 가능) -->
    <TextBox x:Name="txtInput" />
    <!--
        → 코드 비하인드에서 txtInput.Text 로 접근 가능
        → WinForms의 컨트롤 Name 속성과 동일
        → MVVM에서는 x:Name을 거의 사용하지 않음!
    -->

    <!-- x:Key: 리소스에 이름(키)을 부여 -->
    <Window.Resources>
        <SolidColorBrush x:Key="PrimaryBrush" Color="#2196F3" />
        <!--
            → {StaticResource PrimaryBrush}로 참조 가능
            → 뒤에서 자세히 설명
        -->
    </Window.Resources>
</Window>
```

---

## 4. 자주 쓰는 컨트롤과 예제

### 4.1 TextBlock — 텍스트 표시 (WinForms의 Label)

```xml
<!-- 기본 TextBlock -->
<TextBlock Text="안녕하세요!" />

<!-- 다양한 속성 -->
<TextBlock Text="스타일 적용 텍스트"
           FontSize="18"
           FontWeight="Bold"
           Foreground="DarkBlue"
           TextWrapping="Wrap"
           TextAlignment="Center" />
<!--
    TextWrapping="Wrap": 텍스트가 길면 자동 줄바꿈
    (WinForms Label은 AutoSize=true로 크기를 키우거나, 수동으로 줄바꿈해야 했음)
-->

<!-- 인라인 서식 (Run으로 부분별 스타일 적용) -->
<TextBlock>
    <Run Text="이름: " FontWeight="Bold" />
    <Run Text="홍길동" Foreground="Red" />
    <LineBreak />
    <Run Text="나이: " FontWeight="Bold" />
    <Run Text="25세" Foreground="Blue" />
</TextBlock>
```

### 4.2 TextBox — 텍스트 입력 (WinForms의 TextBox)

```xml
<!-- 기본 TextBox -->
<TextBox Text="입력하세요" />

<!-- 여러 줄 입력 (WinForms: Multiline = true) -->
<TextBox AcceptsReturn="True"
         TextWrapping="Wrap"
         VerticalScrollBarVisibility="Auto"
         Height="100" />
<!--
    AcceptsReturn="True": Enter 키로 줄바꿈 가능 (WinForms의 Multiline)
    VerticalScrollBarVisibility="Auto": 내용이 많으면 스크롤바 표시
-->

<!-- 비밀번호 입력은 PasswordBox 사용 (TextBox와 별도 컨트롤!) -->
<PasswordBox PasswordChar="*" MaxLength="20" />

<!-- 읽기 전용 TextBox -->
<TextBox Text="수정 불가" IsReadOnly="True" Background="LightGray" />

<!-- 데이터 바인딩과 함께 (MVVM에서의 사용법) -->
<TextBox Text="{Binding UserName, UpdateSourceTrigger=PropertyChanged}" />
<!--
    UpdateSourceTrigger=PropertyChanged:
    키를 누를 때마다 바인딩 소스(ViewModel)에 값을 전달
    기본값은 LostFocus (포커스를 잃을 때 전달)
-->
```

### 4.3 Button — 버튼 (WinForms의 Button)

```xml
<!-- 기본 텍스트 버튼 -->
<Button Content="클릭하세요" Click="Button_Click" />

<!-- WPF Button의 강력한 점: Content에 어떤 요소든 넣을 수 있음! -->
<!-- WinForms에서는 Button.Text에 문자열만 가능했지만... -->
<Button Width="150" Height="50">
    <Button.Content>
        <StackPanel Orientation="Horizontal">
            <!-- 아이콘 + 텍스트 버튼 -->
            <TextBlock Text="&#x1F4BE;" FontSize="20" Margin="0,0,8,0" />
            <TextBlock Text="저장하기" VerticalAlignment="Center" FontSize="14" />
        </StackPanel>
    </Button.Content>
</Button>

<!-- Command 바인딩 (MVVM 방식) -->
<Button Content="저장" Command="{Binding SaveCommand}" />
<!--
    Click 이벤트 핸들러 대신 Command 바인딩 사용
    자세한 내용은 04-mvvm-pattern.md에서 설명
-->

<!-- 비활성화 조건 자동 연결 -->
<Button Content="삭제"
        Command="{Binding DeleteCommand}"
        IsEnabled="{Binding CanDelete}" />
```

### 4.4 ListBox — 목록 표시 (WinForms의 ListBox)

```xml
<!-- 정적 항목 -->
<ListBox>
    <ListBoxItem Content="사과" />
    <ListBoxItem Content="바나나" />
    <ListBoxItem Content="포도" />
</ListBox>

<!-- 데이터 바인딩 (일반적인 사용법) -->
<ListBox ItemsSource="{Binding Fruits}"
         SelectedItem="{Binding SelectedFruit}"
         DisplayMemberPath="Name" />
<!--
    ItemsSource: 목록 데이터 소스 (WinForms의 DataSource와 유사)
    SelectedItem: 선택된 항목과 바인딩
    DisplayMemberPath: 표시할 속성 이름 (WinForms의 DisplayMember와 동일)
-->

<!-- 커스텀 항목 템플릿 (WinForms에서는 불가능했던 것!) -->
<ListBox ItemsSource="{Binding Products}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <!-- 각 항목을 커스텀 레이아웃으로 표시 -->
            <StackPanel Orientation="Horizontal" Margin="5">
                <TextBlock Text="{Binding Name}"
                           FontWeight="Bold"
                           Width="100" />
                <TextBlock Text="{Binding Price, StringFormat={}{0:N0}원}"
                           Foreground="Green" />
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

### 4.5 ComboBox — 드롭다운 목록 (WinForms의 ComboBox)

```xml
<!-- 정적 항목 -->
<ComboBox SelectedIndex="0">
    <ComboBoxItem Content="서울" />
    <ComboBoxItem Content="부산" />
    <ComboBoxItem Content="대구" />
</ComboBox>

<!-- 데이터 바인딩 -->
<ComboBox ItemsSource="{Binding Cities}"
          SelectedItem="{Binding SelectedCity}"
          DisplayMemberPath="CityName" />
<!--
    WinForms와 거의 동일한 속성 이름:
    ItemsSource ↔ DataSource
    DisplayMemberPath ↔ DisplayMember
    SelectedValuePath ↔ ValueMember
-->

<!-- 편집 가능한 ComboBox (WinForms의 DropDownStyle.DropDown) -->
<ComboBox IsEditable="True"
          Text="{Binding SearchText}"
          ItemsSource="{Binding Suggestions}" />
```

### 4.6 DataGrid — 데이터 그리드 (WinForms의 DataGridView)

```xml
<!-- 자동 열 생성 (바인딩된 객체의 모든 속성을 열로 표시) -->
<DataGrid ItemsSource="{Binding Employees}"
          AutoGenerateColumns="True"
          IsReadOnly="True" />

<!-- 수동 열 정의 (실무에서 더 많이 사용) -->
<DataGrid ItemsSource="{Binding Employees}"
          AutoGenerateColumns="False"
          CanUserAddRows="False"
          CanUserDeleteRows="False"
          SelectionMode="Single"
          SelectedItem="{Binding SelectedEmployee}">
    <DataGrid.Columns>
        <!-- 텍스트 열 -->
        <DataGridTextColumn Header="사번"
                            Binding="{Binding Id}"
                            Width="80"
                            IsReadOnly="True" />

        <!-- 텍스트 열 -->
        <DataGridTextColumn Header="이름"
                            Binding="{Binding Name}"
                            Width="120" />

        <!-- 체크박스 열 -->
        <DataGridCheckBoxColumn Header="재직"
                                Binding="{Binding IsActive}"
                                Width="60" />

        <!-- 콤보박스 열 -->
        <DataGridComboBoxColumn Header="부서"
                                SelectedItemBinding="{Binding Department}"
                                Width="100" />

        <!-- 커스텀 템플릿 열 -->
        <DataGridTemplateColumn Header="급여" Width="120">
            <DataGridTemplateColumn.CellTemplate>
                <DataTemplate>
                    <TextBlock Text="{Binding Salary, StringFormat={}{0:N0}원}"
                               HorizontalAlignment="Right"
                               Padding="5,0" />
                </DataTemplate>
            </DataGridTemplateColumn.CellTemplate>
        </DataGridTemplateColumn>
    </DataGrid.Columns>
</DataGrid>
```

### 4.7 CheckBox, RadioButton

```xml
<!-- CheckBox (WinForms의 CheckBox와 동일) -->
<CheckBox Content="동의합니다"
          IsChecked="{Binding IsAgreed}" />

<!-- RadioButton (WinForms의 RadioButton과 동일) -->
<!-- GroupName으로 그룹 구분 (WinForms에서는 GroupBox 안에 넣어서 구분) -->
<StackPanel>
    <TextBlock Text="성별:" FontWeight="Bold" Margin="0,0,0,5" />
    <RadioButton Content="남성" GroupName="Gender"
                 IsChecked="{Binding IsMale}" />
    <RadioButton Content="여성" GroupName="Gender"
                 IsChecked="{Binding IsFemale}" />
</StackPanel>
```

---

## 5. 레이아웃 시스템

WinForms에서는 컨트롤의 `Location` 속성으로 **절대 좌표**를 지정했습니다.
WPF에서는 **레이아웃 패널(Panel)** 을 사용해 상대적으로 배치합니다.

### 5.1 Grid — 가장 많이 쓰는 레이아웃 (표 형태 배치)

Grid는 **행(Row)과 열(Column)** 로 영역을 나누고,
자식 요소를 `Grid.Row`와 `Grid.Column`으로 배치합니다.

```xml
<Grid>
    <!-- 행 정의 -->
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto" />    <!-- 내용에 맞게 높이 자동 조정 -->
        <RowDefinition Height="50" />      <!-- 고정 50 DIP -->
        <RowDefinition Height="*" />       <!-- 나머지 공간 전부 차지 -->
        <RowDefinition Height="2*" />      <!-- 나머지 공간의 2배 비율 -->
    </Grid.RowDefinitions>

    <!-- 열 정의 -->
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="100" />   <!-- 고정 100 DIP -->
        <ColumnDefinition Width="*" />     <!-- 나머지 공간 전부 차지 -->
    </Grid.ColumnDefinitions>

    <!-- 자식 요소 배치 (Grid.Row, Grid.Column은 0부터 시작) -->
    <TextBlock Grid.Row="0" Grid.Column="0" Text="이름:" />
    <TextBox   Grid.Row="0" Grid.Column="1" />

    <TextBlock Grid.Row="1" Grid.Column="0" Text="이메일:" />
    <TextBox   Grid.Row="1" Grid.Column="1" />

    <!-- 여러 열에 걸쳐 배치 -->
    <Button Grid.Row="2" Grid.Column="0" Grid.ColumnSpan="2"
            Content="저장" />
</Grid>
```

```
Grid.RowDefinitions / Grid.ColumnDefinitions 크기 지정 방법:

  Auto   → 내용물에 맞게 자동 조정
  100    → 고정 크기 (100 DIP)
  *      → 남은 공간을 균등 분배 (비율: 1)
  2*     → 남은 공간의 2배 비율
  3*     → 남은 공간의 3배 비율

  예: *, 2*, 3* 이면 총 6등분 → 1/6, 2/6, 3/6 비율로 분배
```

### 실전 예제: 로그인 폼

```xml
<!-- 로그인 폼 레이아웃 예제 -->
<Grid Margin="20">
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto" />   <!-- 제목 -->
        <RowDefinition Height="Auto" />   <!-- 아이디 -->
        <RowDefinition Height="Auto" />   <!-- 비밀번호 -->
        <RowDefinition Height="Auto" />   <!-- 버튼 -->
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="80" />   <!-- 라벨 -->
        <ColumnDefinition Width="*" />    <!-- 입력 필드 -->
    </Grid.ColumnDefinitions>

    <!-- 제목 (2열에 걸쳐 가운데 정렬) -->
    <TextBlock Grid.Row="0" Grid.Column="0" Grid.ColumnSpan="2"
               Text="로그인"
               FontSize="24" FontWeight="Bold"
               HorizontalAlignment="Center"
               Margin="0,0,0,20" />

    <!-- 아이디 -->
    <TextBlock Grid.Row="1" Grid.Column="0"
               Text="아이디:"
               VerticalAlignment="Center" />
    <TextBox Grid.Row="1" Grid.Column="1"
             Margin="0,0,0,10" Padding="5,3" />

    <!-- 비밀번호 -->
    <TextBlock Grid.Row="2" Grid.Column="0"
               Text="비밀번호:"
               VerticalAlignment="Center" />
    <PasswordBox Grid.Row="2" Grid.Column="1"
                 Margin="0,0,0,10" Padding="5,3" />

    <!-- 로그인 버튼 -->
    <Button Grid.Row="3" Grid.Column="0" Grid.ColumnSpan="2"
            Content="로그인"
            Height="35" FontSize="14"
            Margin="0,10,0,0" />
</Grid>
```

### 5.2 StackPanel — 순차 배치 (가로 또는 세로)

StackPanel은 자식 요소를 **한 방향으로 차례대로** 쌓습니다.
WinForms의 FlowLayoutPanel과 비슷하지만, 한 방향만 지원합니다.

```xml
<!-- 세로 방향 (기본값) -->
<StackPanel Orientation="Vertical">
    <TextBlock Text="첫 번째" />
    <TextBlock Text="두 번째" />
    <TextBlock Text="세 번째" />
</StackPanel>
<!--
    결과:
    ┌──────────────┐
    │ 첫 번째       │
    │ 두 번째       │
    │ 세 번째       │
    └──────────────┘
-->

<!-- 가로 방향 -->
<StackPanel Orientation="Horizontal">
    <Button Content="파일" Margin="2" />
    <Button Content="편집" Margin="2" />
    <Button Content="보기" Margin="2" />
    <Button Content="도움말" Margin="2" />
</StackPanel>
<!--
    결과:
    ┌──────────────────────────────────────┐
    │ [파일] [편집] [보기] [도움말]          │
    └──────────────────────────────────────┘
-->

<!-- 실전: 입력 폼 (StackPanel 중첩) -->
<StackPanel Margin="20" Width="300">
    <TextBlock Text="이름:" FontWeight="Bold" Margin="0,0,0,5" />
    <TextBox Margin="0,0,0,15" />

    <TextBlock Text="이메일:" FontWeight="Bold" Margin="0,0,0,5" />
    <TextBox Margin="0,0,0,15" />

    <StackPanel Orientation="Horizontal" HorizontalAlignment="Right">
        <Button Content="취소" Width="80" Margin="0,0,10,0" />
        <Button Content="저장" Width="80" />
    </StackPanel>
</StackPanel>
```

### 5.3 DockPanel — 도킹 배치

DockPanel은 자식 요소를 **상/하/좌/우에 도킹**시킵니다.
WinForms의 `Dock` 속성과 비슷한 개념입니다.

```xml
<!-- 전형적인 앱 레이아웃: 메뉴바 + 사이드바 + 상태바 + 메인 콘텐츠 -->
<DockPanel LastChildFill="True">
    <!--
        LastChildFill="True" (기본값):
        마지막 자식이 남은 공간을 전부 차지
    -->

    <!-- 상단: 메뉴 -->
    <Menu DockPanel.Dock="Top">
        <MenuItem Header="파일">
            <MenuItem Header="새로 만들기" />
            <MenuItem Header="열기" />
            <Separator />
            <MenuItem Header="종료" />
        </MenuItem>
        <MenuItem Header="편집">
            <MenuItem Header="잘라내기" />
            <MenuItem Header="복사" />
            <MenuItem Header="붙여넣기" />
        </MenuItem>
    </Menu>

    <!-- 하단: 상태바 -->
    <StatusBar DockPanel.Dock="Bottom">
        <StatusBarItem Content="준비" />
    </StatusBar>

    <!-- 왼쪽: 탐색 패널 -->
    <Border DockPanel.Dock="Left" Width="200" Background="LightGray">
        <StackPanel Margin="10">
            <TextBlock Text="탐색" FontWeight="Bold" Margin="0,0,0,10" />
            <Button Content="대시보드" Margin="0,0,0,5" />
            <Button Content="고객 관리" Margin="0,0,0,5" />
            <Button Content="주문 관리" Margin="0,0,0,5" />
        </StackPanel>
    </Border>

    <!-- 나머지: 메인 콘텐츠 (LastChildFill에 의해 남은 공간 전부 차지) -->
    <Grid Background="White" Margin="10">
        <TextBlock Text="여기가 메인 콘텐츠 영역입니다"
                   HorizontalAlignment="Center"
                   VerticalAlignment="Center"
                   FontSize="20" />
    </Grid>
</DockPanel>
```

```
DockPanel 결과:
┌────────────────────────────────────────────┐
│  [파일]  [편집]                    ← Top    │
├──────────┬─────────────────────────────────┤
│ 탐색     │                                 │
│          │    메인 콘텐츠 영역               │
│ [대시보드]│    (LastChildFill)               │
│ [고객관리]│                                 │
│ [주문관리]│                                 │
│  ← Left  │                                 │
├──────────┴─────────────────────────────────┤
│  준비                              ← Bottom │
└────────────────────────────────────────────┘
```

### 5.4 WrapPanel — 자동 줄바꿈 배치

WrapPanel은 자식 요소를 가로로 배치하다가 공간이 부족하면 **자동으로 줄바꿈**합니다.
WinForms의 FlowLayoutPanel(FlowDirection.LeftToRight)과 유사합니다.

```xml
<!-- 태그 목록 같은 UI에 적합 -->
<WrapPanel Margin="10">
    <Border Background="LightBlue" CornerRadius="10" Padding="8,4" Margin="3">
        <TextBlock Text="C#" />
    </Border>
    <Border Background="LightGreen" CornerRadius="10" Padding="8,4" Margin="3">
        <TextBlock Text="WPF" />
    </Border>
    <Border Background="LightCoral" CornerRadius="10" Padding="8,4" Margin="3">
        <TextBlock Text="MVVM" />
    </Border>
    <Border Background="LightYellow" CornerRadius="10" Padding="8,4" Margin="3">
        <TextBlock Text="XAML" />
    </Border>
    <Border Background="Lavender" CornerRadius="10" Padding="8,4" Margin="3">
        <TextBlock Text="데이터 바인딩" />
    </Border>
</WrapPanel>
<!--
    창이 넓을 때:  [C#] [WPF] [MVVM] [XAML] [데이터 바인딩]
    창이 좁을 때:  [C#] [WPF] [MVVM]
                   [XAML] [데이터 바인딩]
-->
```

### 5.5 레이아웃 패널 비교 요약

| 패널 | 설명 | WinForms 대응 | 주요 사용처 |
|---|---|---|---|
| **Grid** | 행/열 기반 표 형태 배치 | TableLayoutPanel | 폼 레이아웃, 복잡한 배치 |
| **StackPanel** | 한 방향으로 순차 배치 | FlowLayoutPanel (한 방향) | 메뉴, 리스트, 간단한 폼 |
| **DockPanel** | 상/하/좌/우 도킹 | Form의 Dock 속성 | 앱의 전체 레이아웃 구조 |
| **WrapPanel** | 자동 줄바꿈 배치 | FlowLayoutPanel | 태그, 썸네일 목록 |
| **Canvas** | 절대 좌표 배치 | Form 기본 (Location) | 그래픽, 드래그앤드롭 |
| **UniformGrid** | 균등 분할 배치 | 없음 | 바둑판, 계산기 버튼 |

---

## 6. 리소스와 스타일

### 6.1 리소스(Resource)란?

WPF에서 **리소스(Resource)** 는 XAML에서 재사용할 수 있는 **공유 객체**입니다.
색상, 브러시, 스타일, 컨버터 등을 한 번 정의하고 여러 곳에서 참조합니다.

```xml
<Window.Resources>
    <!-- 색상 정의 -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="#2196F3" />
    <SolidColorBrush x:Key="DangerBrush" Color="#F44336" />

    <!-- 문자열 상수 -->
    <sys:String x:Key="AppTitle"
                xmlns:sys="clr-namespace:System;assembly=mscorlib">
        내 애플리케이션
    </sys:String>

    <!-- 숫자 상수 -->
    <sys:Double x:Key="DefaultFontSize"
                xmlns:sys="clr-namespace:System;assembly=mscorlib">
        14
    </sys:Double>
</Window.Resources>

<!-- 리소스 사용 -->
<TextBlock Text="{StaticResource AppTitle}"
           FontSize="{StaticResource DefaultFontSize}"
           Foreground="{StaticResource PrimaryBrush}" />

<Button Content="삭제"
        Background="{StaticResource DangerBrush}" />
```

### StaticResource vs DynamicResource

```xml
<!-- StaticResource: 리소스를 한 번만 찾아서 적용 (성능 좋음) -->
<TextBlock Foreground="{StaticResource PrimaryBrush}" />

<!-- DynamicResource: 리소스가 변경되면 자동으로 다시 적용 (테마 전환 등에 사용) -->
<TextBlock Foreground="{DynamicResource PrimaryBrush}" />
```

```
StaticResource: 앱 시작 시 → 리소스 조회 → 값 적용 → 끝 (변경 불가)
DynamicResource: 앱 시작 시 → 리소스 조회 → 값 적용 → 리소스 변경 감시 → 자동 업데이트
```

**일반적인 규칙**: 대부분 `StaticResource`를 사용하고,
테마 전환이 필요한 경우에만 `DynamicResource`를 사용합니다.

### 리소스의 범위 (검색 순서)

```xml
<!-- 리소스는 계층적으로 검색됩니다 (자식 → 부모 → Window → App) -->

<Application.Resources>
    <!-- 앱 전체에서 사용 가능 (가장 넓은 범위) -->
    <SolidColorBrush x:Key="AppBrush" Color="Blue" />
</Application.Resources>

<Window.Resources>
    <!-- 이 Window 안에서만 사용 가능 -->
    <SolidColorBrush x:Key="WindowBrush" Color="Green" />
</Window.Resources>

<Grid>
    <Grid.Resources>
        <!-- 이 Grid 안에서만 사용 가능 (가장 좁은 범위) -->
        <SolidColorBrush x:Key="GridBrush" Color="Red" />
    </Grid.Resources>

    <!-- 여기서 {StaticResource AppBrush}를 쓰면:
         Grid.Resources → 없음
         Window.Resources → 없음
         App.Resources → 찾음! Blue 적용 -->
</Grid>
```

### 6.2 Style — 스타일

WinForms에서는 각 컨트롤의 속성을 **하나하나** 설정해야 했습니다.
WPF Style을 사용하면 **공통 속성을 한 번에** 적용할 수 있습니다.

```csharp
// WinForms: 버튼마다 일일이 설정해야 함
button1.Font = new Font("맑은 고딕", 12);
button1.Height = 35;
button1.BackColor = Color.FromArgb(33, 150, 243);
button1.ForeColor = Color.White;

button2.Font = new Font("맑은 고딕", 12);  // 같은 설정 반복!
button2.Height = 35;
button2.BackColor = Color.FromArgb(33, 150, 243);
button2.ForeColor = Color.White;
```

```xml
<!-- WPF: Style을 한 번 정의하면 모든 버튼에 자동 적용 -->
<Window.Resources>
    <!-- TargetType만 지정하고 x:Key를 생략하면 해당 타입의 모든 컨트롤에 자동 적용! -->
    <Style TargetType="Button">
        <Setter Property="FontSize" Value="14" />
        <Setter Property="Height" Value="35" />
        <Setter Property="Background" Value="#2196F3" />
        <Setter Property="Foreground" Value="White" />
        <Setter Property="Padding" Value="15,5" />
        <Setter Property="Margin" Value="5" />
    </Style>
</Window.Resources>

<!-- 이 Window 안의 모든 Button에 위 스타일이 자동 적용됨! -->
<StackPanel>
    <Button Content="버튼 1" />  <!-- 자동으로 파란 배경, 흰 글씨 -->
    <Button Content="버튼 2" />  <!-- 자동으로 파란 배경, 흰 글씨 -->
    <Button Content="버튼 3" />  <!-- 자동으로 파란 배경, 흰 글씨 -->
</StackPanel>
```

### 이름이 있는 스타일 (선택적 적용)

```xml
<Window.Resources>
    <!-- x:Key가 있으면 명시적으로 적용해야 함 -->
    <Style x:Key="DangerButton" TargetType="Button">
        <Setter Property="Background" Value="#F44336" />
        <Setter Property="Foreground" Value="White" />
        <Setter Property="FontWeight" Value="Bold" />
    </Style>

    <Style x:Key="SuccessButton" TargetType="Button">
        <Setter Property="Background" Value="#4CAF50" />
        <Setter Property="Foreground" Value="White" />
    </Style>
</Window.Resources>

<StackPanel>
    <Button Content="삭제" Style="{StaticResource DangerButton}" />
    <Button Content="저장" Style="{StaticResource SuccessButton}" />
    <Button Content="일반 버튼" />  <!-- 스타일 없음 (기본 모양) -->
</StackPanel>
```

### 스타일 상속 (BasedOn)

```xml
<Window.Resources>
    <!-- 기본 버튼 스타일 -->
    <Style x:Key="BaseButton" TargetType="Button">
        <Setter Property="FontSize" Value="14" />
        <Setter Property="Height" Value="35" />
        <Setter Property="Padding" Value="15,5" />
        <Setter Property="Margin" Value="5" />
    </Style>

    <!-- 기본 스타일을 상속받아 확장 -->
    <Style x:Key="PrimaryButton" TargetType="Button"
           BasedOn="{StaticResource BaseButton}">
        <!-- BaseButton의 모든 속성을 상속 + 추가 설정 -->
        <Setter Property="Background" Value="#2196F3" />
        <Setter Property="Foreground" Value="White" />
    </Style>

    <Style x:Key="DangerButton" TargetType="Button"
           BasedOn="{StaticResource BaseButton}">
        <Setter Property="Background" Value="#F44336" />
        <Setter Property="Foreground" Value="White" />
    </Style>
</Window.Resources>
```

### 6.3 ControlTemplate — 컨트롤 외형 완전 재정의

WinForms에서는 Button의 외형을 바꾸려면 Owner Draw를 해야 했습니다.
WPF에서는 `ControlTemplate`으로 컨트롤의 **시각적 구조를 완전히 재정의**할 수 있습니다.

```xml
<Window.Resources>
    <!-- 둥근 모서리 버튼 스타일 -->
    <Style x:Key="RoundedButton" TargetType="Button">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <!-- 버튼의 시각적 구조를 처음부터 정의 -->
                    <Border Background="{TemplateBinding Background}"
                            CornerRadius="15"
                            Padding="{TemplateBinding Padding}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="1">
                        <!--
                            TemplateBinding: 버튼에 설정된 속성 값을 가져옴
                            예: <Button Background="Blue"> → Border의 Background에 Blue 적용
                        -->
                        <ContentPresenter HorizontalAlignment="Center"
                                          VerticalAlignment="Center" />
                        <!--
                            ContentPresenter: Button.Content의 내용을 표시하는 위치
                        -->
                    </Border>

                    <!-- 트리거: 마우스 오버 시 색상 변경 -->
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter Property="Background" Value="#1976D2" />
                        </Trigger>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter Property="Background" Value="#0D47A1" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
        <!-- 기본 속성값 -->
        <Setter Property="Background" Value="#2196F3" />
        <Setter Property="Foreground" Value="White" />
        <Setter Property="Padding" Value="20,8" />
        <Setter Property="FontSize" Value="14" />
    </Style>
</Window.Resources>

<!-- 사용 -->
<Button Content="둥근 버튼" Style="{StaticResource RoundedButton}" />
```

---

## 7. WinForms 개발자가 헷갈리는 점들

### 7.1 Label vs TextBlock

```
WinForms의 Label → WPF에서는 TextBlock 사용 (대부분의 경우)

WPF에도 Label 컨트롤이 있지만:
- Label은 ContentControl (내부에 어떤 요소든 넣을 수 있음)
- TextBlock은 순수 텍스트 전용 (더 가볍고 빠름)

→ 단순 텍스트 표시에는 항상 TextBlock 사용!
```

```xml
<!-- 대부분의 경우 TextBlock 사용 -->
<TextBlock Text="이름:" />

<!-- Label은 Access Key (단축키) 기능이 필요할 때 사용 -->
<Label Content="_이름:" Target="{Binding ElementName=txtName}" />
<!-- Alt+이 누르면 txtName으로 포커스 이동 -->
<TextBox x:Name="txtName" />
```

### 7.2 Content vs Text

```xml
<!-- WPF에서 가장 헷갈리는 점 중 하나! -->

<!-- TextBlock, TextBox: Text 속성 사용 -->
<TextBlock Text="표시할 텍스트" />
<TextBox Text="입력할 텍스트" />

<!-- Button, Label, CheckBox 등: Content 속성 사용 -->
<Button Content="버튼 텍스트" />
<Label Content="라벨 텍스트" />
<CheckBox Content="체크박스 텍스트" />

<!--
    이유: Content는 문자열뿐만 아니라 어떤 UI 요소든 넣을 수 있음!
    Text는 순수 문자열만 가능
-->
<Button>
    <Button.Content>
        <!-- 버튼 안에 이미지와 텍스트를 함께 넣기 -->
        <StackPanel Orientation="Horizontal">
            <Image Source="icon.png" Width="16" Height="16" />
            <TextBlock Text="저장" Margin="5,0,0,0" />
        </StackPanel>
    </Button.Content>
</Button>
```

### 7.3 Margin vs Padding vs 왜 Location이 없는가?

```xml
<!--
    WinForms: Location 속성으로 절대 좌표 지정
    WPF: Margin으로 상대적 여백 지정

    Margin: 컨트롤 바깥쪽 여백 (다른 요소와의 간격)
    Padding: 컨트롤 안쪽 여백 (테두리와 내용 사이의 간격)
-->

<!-- Margin: 왼쪽, 위, 오른쪽, 아래 순서 -->
<Button Content="버튼" Margin="10,5,10,5" />   <!-- 왼10, 위5, 오10, 아5 -->
<Button Content="버튼" Margin="10,5" />         <!-- 좌우10, 상하5 (축약) -->
<Button Content="버튼" Margin="10" />           <!-- 상하좌우 모두 10 (축약) -->

<!-- Padding -->
<Button Content="버튼" Padding="20,10" />       <!-- 좌우20, 상하10의 내부 여백 -->

<!--
    ┌─────────── Margin ──────────────┐
    │   ┌────── Border ────────┐      │
    │   │  ┌─ Padding ──┐     │      │
    │   │  │   Content   │     │      │
    │   │  └─────────────┘     │      │
    │   └──────────────────────┘      │
    └─────────────────────────────────┘
-->
```

### 7.4 Visibility 속성의 3가지 값

```xml
<!--
    WinForms: Visible = true/false (2가지)
    WPF: Visibility = Visible/Hidden/Collapsed (3가지!)
-->

<StackPanel>
    <TextBlock Text="보이는 요소" Visibility="Visible" />
    <!-- Visible: 보이고, 공간도 차지함 (기본값) -->

    <TextBlock Text="숨긴 요소 (공간 유지)" Visibility="Hidden" />
    <!-- Hidden: 안 보이지만, 공간은 그대로 차지함 (WinForms의 Visible=false와 유사) -->

    <TextBlock Text="접힌 요소 (공간 없음)" Visibility="Collapsed" />
    <!-- Collapsed: 안 보이고, 공간도 차지하지 않음 (레이아웃에서 완전히 제거) -->
</StackPanel>

<!--
    Visible:    [보이는 요소]
    Hidden:     [              ]  ← 빈 공간이 있음
    Collapsed:  (아무것도 없음)   ← 공간 자체가 없음
-->
```

### 7.5 이벤트가 동작하는 방식이 다르다

```xml
<!--
    WinForms: 이벤트가 해당 컨트롤에서만 발생
    WPF: 이벤트가 트리를 따라 이동 (Routed Events)

    - Bubbling (버블링): 자식 → 부모 방향으로 이벤트 전파
    - Tunneling (터널링): 부모 → 자식 방향으로 이벤트 전파 (Preview~ 접두사)
    - Direct: 해당 컨트롤에서만 발생 (WinForms와 동일)
-->

<!-- 예: Grid 안의 Button을 클릭해도 Grid에서 이벤트를 잡을 수 있음 -->
<Grid MouseLeftButtonDown="Grid_MouseLeftButtonDown">
    <Button Content="클릭" />
    <!-- Button 클릭 → Button에서 이벤트 발생 → Grid로 버블링 -->
</Grid>

<!-- Tunneling 이벤트 (Preview 접두사) -->
<TextBox PreviewKeyDown="TextBox_PreviewKeyDown" />
<!-- PreviewKeyDown: 키가 눌리기 전에 부모에서 자식으로 내려오는 이벤트 -->
<!-- KeyDown: 키가 눌린 후 자식에서 부모로 올라가는 이벤트 -->
```

### 7.6 WinForms 컨트롤과 WPF 컨트롤 이름 매핑

| WinForms | WPF | 비고 |
|---|---|---|
| `Label` | `TextBlock` | 단순 텍스트에는 TextBlock 사용 |
| `TextBox` | `TextBox` | 동일 |
| `Button` | `Button` | Content 속성 사용 (Text 아님) |
| `CheckBox` | `CheckBox` | 동일 |
| `RadioButton` | `RadioButton` | GroupName 속성으로 그룹 구분 |
| `ComboBox` | `ComboBox` | 동일 |
| `ListBox` | `ListBox` | 동일 |
| `DataGridView` | `DataGrid` | 이름이 다름 주의! |
| `PictureBox` | `Image` | 이름이 다름 주의! |
| `Panel` | `Grid` / `Border` | 용도에 따라 선택 |
| `GroupBox` | `GroupBox` | 동일 |
| `TabControl` | `TabControl` | 동일 |
| `ToolStrip` | `ToolBar` | 이름이 다름 |
| `StatusStrip` | `StatusBar` | 이름이 다름 |
| `MenuStrip` | `Menu` | 이름이 다름 |
| `ProgressBar` | `ProgressBar` | 동일 |
| `NumericUpDown` | 없음 (직접 구현 필요) | WPF에 기본 제공 안됨! |
| `RichTextBox` | `RichTextBox` | API가 완전히 다름 |

---

> **핵심 요약**
>
> 1. **XAML은 C# 객체를 XML로 표현한 것**입니다. 모든 태그는 클래스, 모든 속성은 프로퍼티입니다.
> 2. **레이아웃은 패널(Panel) 시스템**을 사용합니다. Grid가 가장 많이 쓰이며, 절대 좌표 배치는 지양합니다.
> 3. **Style로 공통 속성을 일괄 적용**할 수 있습니다. CSS처럼 생각하면 됩니다.
> 4. **ControlTemplate으로 컨트롤 외형을 완전히 재정의**할 수 있습니다.
> 5. WinForms와 **컨트롤 이름, 속성 이름이 다른 경우**가 있으니 매핑표를 참고하세요.
