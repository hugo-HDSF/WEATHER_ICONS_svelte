<script lang="ts">

    //selected city
    import {onMount} from 'svelte';

    let cityOptions = [];
    let inputCityName = '';

    onMount(async () => {
        const response = await fetch('../data/cities-fr.json');
        const data = await response.json();
        cityOptions = data.map(city => ({
            name: city.nm,
            value: city.id.toString(),
            lat: city.lat,
            lon: city.lon,
            keywords: '',
            meta: {healthy: true}
        }));
        if (cityOptions.length > 0) {
            inputCityName = cityOptions[0].value;
        }
    });

    // city selection
    let city;

    //fetch weather data when the selected city changes
    $: {
        city = cityOptions.find(option => option.value === inputCityName);
        if (city) {
            fetchWeatherData(city);
        }
    }

    //date formatting for ->
    // - current weather
    const current_weather_fulldate_option = {weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'};
    // - forecast weekday
    const forecast_weather_weekday_option = {weekday: 'short'};
    // - forecast 3h hour
    const forecast3h_weather_time_option = {hour: 'numeric', hour12: false };
    // - forecast day/month
    const forecast3h_weather_date_options = {day: 'numeric', month: 'numeric'};

    //weather conditions
    import {writable} from 'svelte/store';

    //loading state
    let loading = false;

    //weather data to be displayed
    let currentWeather;
    let forecastWeather;
    let currentWeatherData;
    let forecastWeatherData;

    // Create a writable store to hold the weather data
    //(forcast as iterable array of objects)
    // (current as object) mapped afterwards in the fetchWeatherData function to an iterable array of objects
    let weatherData = writable({
        current: null,
        forecasts: [],
    });

    //subscribe to the weather data store changes
    weatherData.subscribe(value => {
        currentWeather = value.current;
        forecastWeather = value.forecasts;
    });

    //fetch weather data from the openweathermap API
    async function fetchWeatherData(city) {
        //set loading state to true
        loading = true;
        // Construct the API URL
        const currentWeatherUrl = `https://api.openweathermap.org/data/2.5/weather?lat=${encodeURIComponent(city.lat)}&lon=${encodeURIComponent(city.lon)}&exclude=minutely,hourly,alerts&units=metric&lang=fr&appid=b5c8c90ecdc66b2a65d47c59cf1f32fd`;
        const forecastWeatherUrl = `https://api.openweathermap.org/data/2.5/forecast?lat=${encodeURIComponent(city.lat)}&lon=${encodeURIComponent(city.lon)}&exclude=minutely,hourly,alerts&units=metric&lang=fr&appid=b5c8c90ecdc66b2a65d47c59cf1f32fd`;

        //https://openweathermap.org/img/wn/10d@2x.png
        try {
            // Fetch the data from the API
            const currentResponse = await fetch(currentWeatherUrl);
            currentWeatherData = await currentResponse.json();

            const forecastResponse = await fetch(forecastWeatherUrl);
            forecastWeatherData = await forecastResponse.json();

            weatherData.set({
                current: {
                    // map the current weather data to convert the object of arrays into an array of objects (iterable for #each)
                    temp_min: currentWeatherData.main.temp_min.toPrecision(2),
                    temp_max: currentWeatherData.main.temp_max.toPrecision(2),
                    temp: currentWeatherData.main.temp.toPrecision(2),
                    weather: currentWeatherData.weather[0].description,
                    dt: new Date(currentWeatherData.dt * 1000),
                    icon: currentWeatherData.weather[0].icon,
                },
                forecasts: Object.entries(
                    forecastWeatherData.list
                        // Filter out the 3h period forecasts for the current date (we only want the next 5 days)
                        .filter(_3hforecast => {
                            let currentDate = new Date().setHours(0, 0, 0, 0);
                            let forecastDate = new Date(_3hforecast.dt * 1000).setHours(0, 0, 0, 0);
                            return forecastDate > currentDate;
                        })
                        // Group the forecasts by date into iterable accumulator arrays (one array per day)
                        .reduce((acc, forecast) => {
                            const date = new Date(forecast.dt * 1000).toDateString();
                            if (!acc[date]) {
                                acc[date] = [];
                            }
                            acc[date].push(forecast);
                            return acc;
                        }, {})
                    // map the forecasts for the 5 next days to convert the object of arrays into an array of objects (iterable for #each)
                ).map(([date, forecast]) => {
                    // if the last array has less than 4 forecasts, get the last element, else, get the 4th (3h * [4] = 12h noon)
                    let forecastIndex = forecast.length < 4 ? forecast.length - 1 : 3;
                    return {
                        date,
                        //get the min and max temp for the day
                        temp_min: Math.min(...forecast.map(forecast => parseFloat(forecast.main.temp_min.toPrecision(2)))),
                        temp_max: Math.max(...forecast.map(forecast => parseFloat(forecast.main.temp_max.toPrecision(2)))),
                        temp: forecast[forecastIndex].main.temp.toPrecision(2),
                        //get the 4th weather icon for the day => (3h * 4 ) = 12h (noon) (show the weather at noon, usually the warmest time of the day and the most relevant)
                        icon: forecast[forecastIndex].weather[0].icon,
                        dt: new Date(forecast[0].dt * 1000),
                    }
                })

            });
            //set loading state to false
            loading = false;
            currentWeatherData = currentWeatherData.list;
            forecastWeatherData = forecastWeatherData.list;
        } catch
            (error) {
            console.error('Error fetching weather data:', error);
        }
    }

</script>

<main>
    <div>
        <div class="card">
            <div class="title">
                <svg class="select-city"></svg>
                <span>Selectionner votre ville</span>
            </div>
            <label class="label">
                <select class="select" bind:value={inputCityName}>
                    {#each cityOptions as option (option.value)}
                        <option value={option.value}>{option.name}</option>
                    {/each}
                </select>
            </label>
        </div>


        {#if loading}
            <div class="wrapper">
                <div class="lds-spinner">
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                    <div></div>
                </div>
            </div>


        {:else if currentWeather}
            <!-- today's current weather -->
            <div class="current-weather">
                <h2>{currentWeather.dt.toLocaleDateString('fr-FR', current_weather_fulldate_option)}</h2>
                <h1>{city.name}</h1>
                <span class="main-temp">{currentWeather.temp}°</span>
                <span class="weather-state">{currentWeather.weather}</span>
                <div class="temps">
                    <i class="high-temp"></i>
                    <p>{currentWeather.temp_max}°</p>
                    <span></span>
                    <i class="low-temp"></i>
                    <p>{currentWeather.temp_min}°</p>
                </div>
            </div>


            <!-- forecasts each 3h -->
            <div class="card">
                <div class="title">
                    <svg class="calendar"></svg>
                    <span>prévisions toutes les 3 heures</span>
                </div>
                <div class="forecasts3h">
                    <!-- rightnow's forecast -->
                    <div class="forecast3h">
                        <p>{currentWeather.dt.toLocaleDateString('fr-FR',forecast3h_weather_date_options)}</p>
                        <p>Maint.</p>
                        <p><img class="weather-icon" src="images/{currentWeather.icon}.svg" alt=""></p>
                        <p>{currentWeather.temp}°</p>
                    </div>
                    <!-- next 4 days forecast each 3h -->
                    {#each forecastWeatherData as forecast3h}
                        <div class="forecast3h">
                            <p>{new Date(forecast3h.dt * 1000).toLocaleDateString('fr-FR',forecast3h_weather_date_options)}</p>
                            <p>{new Date(forecast3h.dt * 1000).toLocaleTimeString('fr-FR',forecast3h_weather_time_option)}</p>
                            <p><img class="weather-icon" src="images/{forecast3h.weather[0].icon}.svg" alt=""></p>
                            <p>{forecast3h.main.temp.toPrecision(2)}°</p>
                        </div>
                    {/each}
                </div>
            </div>


            <!-- forecasts for the next 5 days -->
            <div class="card">
                <div class="title">
                    <svg class="calendar"></svg>
                    <span>prévisions sur 5 jours</span>
                </div>
                <!-- today's forecast -->
                <div class="forecast">
                    <div class="forecast-day">
                        <p>Auj.</p>
                    </div>
                    <div class="forecast-weather">
                        <p>Maint.</p>
                        <p><img class="weather-icon" src="images/{currentWeather.icon}.svg" alt=""></p>
                    </div>
                    <div class="forecast-temps">
                        <p>{currentWeather.temp_min}°</p>
                        <span></span>
                        <p>{currentWeather.temp_max}°</p>
                    </div>
                </div>
                <!-- next 4 days forecast -->
                {#each forecastWeather as forecast}
                    <div class="forecast">
                        <div class="forecast-day">
                            <p>{forecast.dt.toLocaleDateString('fr-FR', forecast_weather_weekday_option)}</p>
                        </div>
                        <div class="forecast-weather">
                            <p><img class="weather-icon" src="images/{forecast.icon}.svg" alt=""></p>
                        </div>
                        <div class="forecast-temps">
                            <p>{forecast.temp_min}°</p>
                            <span></span>
                            <p>{forecast.temp_max}°</p>
                        </div>
                    </div>
                {/each}
            </div>
        {/if}
    </div>
</main>

<style>
</style>
