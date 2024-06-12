MVVM 아키텍처는 API 호출을 할 때 가장 유용하게 쓰이는 아키텍처 패턴이다.

## **앱의 진화 : 아키텍처, MVVM 패턴과 LiveData**

<aside>
✔️ 다양한 Data 변화에 효율적으로 대응하기 위해 MVVM 패턴을 적용해 보아요.

</aside>

- **MVVM** (**Model**-**View**-**ViewModel**)  패턴이란?
    - **Model**: 데이터와 비즈니스 로직(핵심 기능과 연산)을 처리. 데이터베이스, 네트워크 서비스, 기타 비즈니스 로직이 해당.
    - **View**: 사용자 인터페이스(UI). 사용자에게 정보를 표시하고 사용자의 입력을 받는 것
    - **ViewModel**: View와 Model 사이의 매개체 역할. View에 데이터를 전달하고, View로부터의 입력을 처리
    - **장점**
        - **모듈성 및 유지 보수성**: View와 Model 사이의 의존성이 낮아져 코드의 재사용성과 유지 보수가 편해짐
        - **테스트 용이성**: ViewModel은 UI로부터 분리되어 있어 비즈니스 로직의 테스트가 용이
        - **반응형 UI**: ViewModel을 통해 View와 Model 간의 데이터 바인딩을 구현함으로써, UI가 데이터의 변경에 자동으로 반응
    
    ### ☑️ ViewModel 사용법
    
    1. **ViewModel 클래스 정의**
        1. ViewModel 을 상속하여 class를 선언
        
        ```kotlin
        class HomeViewModel() : ViewModel() {
        ```
        
    2. **Activity 또는 Fragment에서 ViewModel 인스턴스 가져오기**
        
        ```kotlin
        class MyActivity : AppCompatActivity() {
            private val viewModel: HomeViewModel by viewModels()
        
        class MyFragment : Fragment() {
            private val viewModel: HomeViewModel by viewModels()
        ```
        
    3. **Activity의 ViewModel을 Fragment끼리 공유할 경우**
        1. **`activityViewModels`**를 사용하면 프래그먼트가 속한 액티비티의 **`ViewModel`** 인스턴스를 얻을 수 있으며, 액티비티와 여러 프래그먼트 간에 데이터를 공유할 수 있어요
        
        ```kotlin
        class MyActivity : AppCompatActivity() {
            private val viewModel: HomeViewModel by viewModels()
        
        class MyFragment : Fragment() {
            private val viewModel: HomeViewModel by activityViewModels()
        ```
        
- **LiveData란?**
    - **데이터 변경을 관찰할 수 있는 데이터** 홀더 클래스
    - LiveData는 생명주기를 인식
    - 액티비티, 프래그먼트 생명주기 상태에 따라 자동으로 업데이트 됨
    - **장점**
        - **UI와 데이터 상태 동기화**: LiveData를 사용하면 UI 컴포넌트끼리 데이터 상태를 쉽게 동기화할 수 있음
        - **생명주기 인식**: LiveData는 관찰자에게만 활성 상태일 때 데이터 변경을 알림
        - **메모리 누수 방지**: LiveData는 관찰자의 생명 주기를 인식하므로, 관찰자가 활성 상태가 아닐 때는 이벤트를 보내지 않음
    - **사용법**
    
    ```kotlin
    class HomeFragment : Fragment() {
        private val viewModel by activityViewModels<HomeViewModel>()
    ...
        popupBinding.cityList.setOnItemClickListener { _, _, position, _ ->
            **viewModel.setRegion(cities[position])**
            dismiss()
        }
    ```
    
    ```kotlin
    class WeatherViewModel() : ViewModel() {
        private val _regionText = MutableLiveData("서울특별시")
        **val regionText: LiveData<String> = _regionText**
    
        **fun setRegion(city: String) {
            _regionText.value = city
        }**
    }
    ```
    
    ```kotlin
    class WeatherListFragment : Fragment() {
        private val viewModel by activityViewModels<HomeViewModel>()
    ...
        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
    ...
            **viewModel.regionText.observe(viewLifecycleOwner) {
                binding.regionText.text = it
                getWeatherList(currentDate)
            }**
    ```
    
- MVVM 패턴을  WeatherHomdeFragment에 적용하기! **함께 따라해 봅시다 어렵지 않아요 :)**
    
    
    1. **WeatherViewModel 만들기**
        1. getWeather(), getWeatherList()  : repository로부터 weather 정보 받아오는 메소드
        
        ```kotlin
        private val _weatherData = MutableLiveData<WeatherData>()
        val weatherData get() = _weatherData
        
        private val _weatherList = MutableLiveData<**List**<WeatherData>>()
        val weatherList get() = _weatherList
        
        fun getWeather(date: String, city: String = regionText.value ?: "서울특별시") {
            **viewModelScope.launch {**
                runCatching {
                    **repository.getWeather(date, city)**
                }.onSuccess { weatherResponse ->
                    **_weatherData.value = weatherResponse.toWeatherData()**
                }.onFailure { e ->
                    handleException(e)
                }
            }
        }
        
        fun getWeatherList(date: String, count: Int = 20) {
           **viewModelScope.launch {**
                runCatching {
                    val city = regionText.value ?: "서울특별시"
                    **repository.getWeather(date, city, pageNo = count)**
                }.onSuccess { weatherResponse ->
                    **_weatherList.value = weatherResponse.toWeatherList(count)**
                }.onFailure { e ->
                    handleException(e)
                }
            }
        }
        ```
        
            b. regionText : City 선택된 값 저장되는 liveData
        
        ```kotlin
        private val _regionText = MutableLiveData("서울특별시")
        val regionText: LiveData<String> = _regionText
        
        fun setRegion(city: String) {
            _regionText.value = city
        }
        ```
        
    - [코드스니펫] WeatherViewModel
        
        ```kotlin
        import android.util.Log
        import androidx.lifecycle.LiveData
        import androidx.lifecycle.MutableLiveData
        import androidx.lifecycle.ViewModel
        import androidx.lifecycle.viewModelScope
        
        import kotlinx.coroutines.launch
        import retrofit2.HttpException
        import java.io.IOException
        
        private const val TAG = "WeatherViewModel"
        
        class WeatherViewModel(private val repository: WeatherRepository = WeatherRepository()) : ViewModel() {
            private val _regionText = MutableLiveData("서울특별시")
            val regionText: LiveData<String> = _regionText
        
            fun setRegion(city: String) {
                _regionText.value = city
            }
        
            private val _weatherData = MutableLiveData<WeatherData>()
            val weatherData get() = _weatherData
        
            private val _weatherList = MutableLiveData<List<WeatherData>>()
            val weatherList get() = _weatherList
        
            fun getWeather(date: String, city: String = regionText.value ?: "서울특별시") {
                viewModelScope.launch {
                    runCatching {
                        repository.getWeather(date, city)
                    }.onSuccess { weatherResponse ->
                        _weatherData.value = weatherResponse.toWeatherData()
                    }.onFailure { e ->
                        handleException(e)
                    }
                }
            }
        
            fun getWeatherList(date: String, count: Int = 20) {
                viewModelScope.launch {
                    runCatching {
                        val city = regionText.value ?: "서울특별시"
                        repository.getWeather(date, city, pageNo = count)
                    }.onSuccess { weatherResponse ->
                        _weatherList.value = weatherResponse.toWeatherList(count)
                    }.onFailure { e ->
                        handleException(e)
                    }
                }
            }
        
            private fun handleException(e: Throwable) {
                when (e) {
                    is HttpException -> {
                        val errorJsonString = e.response()?.errorBody()?.string()
                        Log.e(TAG, "HTTP error: $errorJsonString")
                    }
        
                    is IOException -> Log.e(TAG, "Network error: $e")
                    else -> Log.e(TAG, "Unexpected error: $e")
                }
            }
        }
        ```
        
    - [코드스니펫] toWeatherList (WeatherModel에 추가)
        
        ```kotlin
        fun toWeatherList(count: Int): List<WeatherData> {
                val list = mutableListOf<WeatherData>()
                val items = response.body.items.item
                val baseTime = items.first().baseTime
                var nextTime = baseTime.nextTime()
                repeat(count) {
                    val subItems = items.filter { it.fcstTime == nextTime }.toList()
                    val weatherData = subItems.toWeatherData()
                    Log.d("WeatherData", "time: $nextTime, data: $weatherData")
                    list.add(weatherData)
                    nextTime = nextTime.nextTime()
                }
                return list
            }
        
            private fun String.nextTime(): String {
                if (this.length != 4) {
                    throw IllegalArgumentException("잘못된 시간 형식")
                }
        
                val hour = this.substring(0, 2).toInt()
                val minute = this.substring(2, 4).toInt()
        
                if (hour !in 0..23 || minute !in 0..59) {
                    throw IllegalArgumentException("잘못된 시간 범위")
                }
        
                val nextHour = if (hour == 23) 0 else hour + 1
                return "%02d%02d".format(nextHour, minute)
            }
        ```
        
    1. WeatherHomeFragment에서 viewModel 참조하기
        1. Fragment끼리 공유하는 용이라 **activityViewModel**로 처리
    
    ```kotlin
    private val viewModel by activityViewModels<WeatherViewModel>()
    ```
    
    - [코드스니펫] viewModel 참조하기
        
        ```kotlin
        private val viewModel by activityViewModels<WeatherViewModel>()
        ```
        
    
    1. viewModel을 활용해 weather 정보 요청, 업데이트 하기
        1. weather 정보 요청하기
    
    ```kotlin
    private fun fetchWeatherData(city: String = "서울특별시") {
        viewModel.getWeather(currentDate, city)
    }
    ```
    
           b.  weatherData LiveData 변화에 따라 UI 업데이트 하기
    
    ```kotlin
    viewModel.**weatherData.observe(**viewLifecycleOwner) { data ->
        with(binding) {
            mainWeatherText.text = data.skyStatus.text
            mainTemperTv.text = data.temperature
            mainRainTv.text = data.rainState.value.toString()
            mainWaterTv.text = data.humidity
            mainWindTv.text = data.windSpeed
            mainRainPercentTv.text = getString(R.string.rain_percent, data.rainPercent)
            rainStatusIv.setImageResource(data.rainState.icon)
            weatherStatusIv.setImageResource(data.skyStatus.colorIcon)
        }
    }
    ```
    
         c. 사용자가 도시를 선택할 수 있는 Popup 구현
    
    ```kotlin
    binding.selectRegionButton.setOnClickListener {
        showCitiesPopup(it)
    }
    
    binding.selectedRegionText.setOnClickListener {
        showCitiesPopup(it)
    }
    ```
    
    ```kotlin
    private fun showCitiesPopup(anchorView: View) {
        val cities = cityList.map { it.name }.toList()
    
        val **popupBinding** = PopupListBinding.inflate(layoutInflater, null, false).also {
            it.cityList.adapter =
                ArrayAdapter(requireContext(), android.R.layout.simple_list_item_1, cities)
        }
    
        PopupWindow(popupBinding.root, 500, LinearLayout.LayoutParams.WRAP_CONTENT, true).apply {
            setBackgroundDrawable(getDrawable(requireContext(), R.drawable.rectangle_background))
            elevation = 10f
            showAsDropDown(anchorView)
            popupBinding.cityList.setOnItemClickListener { _, _, position, _ ->
                val selectedCity = cities[position]
                binding.selectedRegionText.text = selectedCity
                viewModel.setRegion(selectedCity)
                fetchWeatherData(selectedCity)
                dismiss()
            }
        }
    }
    ```
    
    - [코드스니펫] WeatherHomeFragment + viewModel
        
        ```kotlin
        import android.os.Bundle
        import android.view.LayoutInflater
        import android.view.View
        import android.view.ViewGroup
        import android.widget.ArrayAdapter
        import android.widget.LinearLayout
        import android.widget.PopupWindow
        import androidx.appcompat.content.res.AppCompatResources.getDrawable
        import androidx.fragment.app.Fragment
        import androidx.fragment.app.activityViewModels
        import java.text.SimpleDateFormat
        import java.util.Date
        import java.util.Locale
        
        class WeatherHomeFragment : Fragment() {
            private var _binding: FragmentWeatherHomeBinding? = null
            private val binding get() = _binding!!
            private val currentDate by lazy { SimpleDateFormat("yyyyMMdd", Locale.KOREA).format(Date()) }
            **private val viewModel by activityViewModels<WeatherViewModel>()**
        
            override fun onCreateView(
                inflater: LayoutInflater,
                container: ViewGroup?,
                savedInstanceState: Bundle?
            ): View {
                _binding = FragmentWeatherHomeBinding.inflate(inflater, container, false)
                return binding.root
            }
        
            override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
                super.onViewCreated(view, savedInstanceState)
               
                **binding.selectRegionButton.setOnClickListener {
                    showCitiesPopup(it)
                }
        
                binding.selectedRegionText.setOnClickListener {
                    showCitiesPopup(it)
                }
        
                viewModel.weatherData.observe(viewLifecycleOwner) { data ->
                    with(binding) {
                        mainWeatherText.text = data.skyStatus.text
                        mainTemperTv.text = data.temperature
                        mainRainTv.text = data.rainState.value.toString()
                        mainWaterTv.text = data.humidity
                        mainWindTv.text = data.windSpeed
                        mainRainPercentTv.text = getString(R.string.rain_percent, data.rainPercent)
                        rainStatusIv.setImageResource(data.rainState.icon)
                        weatherStatusIv.setImageResource(data.skyStatus.colorIcon)
                    }**
                }
                fetchWeatherData()
            }
        
            private fun showCitiesPopup(anchorView: View) {
                val cities = cityList.map { it.name }.toList()
        
                val popupBinding = PopupListBinding.inflate(layoutInflater, null, false).also {
                    it.cityList.adapter =
                        ArrayAdapter(requireContext(), android.R.layout.simple_list_item_1, cities)
                }
        
                PopupWindow(popupBinding.root, 500, LinearLayout.LayoutParams.WRAP_CONTENT, true).apply {
                    setBackgroundDrawable(getDrawable(requireContext(), R.drawable.rectangle_background))
                    elevation = 10f
                    showAsDropDown(anchorView)
                    popupBinding.cityList.setOnItemClickListener { _, _, position, _ ->
                        val selectedCity = cities[position]
                        binding.selectedRegionText.text = selectedCity
                        viewModel.setRegion(selectedCity)
                        fetchWeatherData(selectedCity)
                        dismiss()
                    }
                }
            }
        
            private fun fetchWeatherData(city: String = "서울특별시") {
                viewModel.getWeather(currentDate, city)
            }
        }
        ```
        
    - [코드스니펫] popup_list.xml
        
        ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <ListView xmlns:android="http://schemas.android.com/apk/res/android"
            android:id="@+id/city_list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"/>
        ```
        
    
    [Screen_recording_20240129_014905.mp4](https://prod-files-secure.s3.us-west-2.amazonaws.com/83c75a39-3aba-4ba4-a792-7aefe4b07895/c3ed8ca8-553a-44a0-9b27-aa2dc0afacde/Screen_recording_20240129_014905.mp4)
