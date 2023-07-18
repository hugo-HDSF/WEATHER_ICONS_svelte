<script lang="ts">

    import {onMount} from 'svelte';

    //selected city
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

    //weather conditions

    import {writable} from 'svelte/store';

    //loading state
    let loading = false;

    //weather data to be displayed
    let current;
    let forecasts;

    // Create a writable store to hold the weather data
    //(forcast as iterable array of objects)
    // (current as object) mapped afterwards in the fetchWeatherData function to an iterable array of objects
    let weatherData = writable({
        current: null,
        forecasts: [],
    });

    //subscribe to the weather data store changes
    weatherData.subscribe(value => {
        current = value.current;
        forecasts = value.forecasts;
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
            const currentData = await currentResponse.json();

            const forecastResponse = await fetch(forecastWeatherUrl);
            const forecastData = await forecastResponse.json();

            weatherData.set({
                current: {
                    // map the current weather data to convert the object of arrays into an array of objects (iterable for #each)
                    temp_min: currentData.main.temp_min.toPrecision(2),
                    temp_max: currentData.main.temp_max.toPrecision(2),
                    weather: currentData.weather[0].description,
                    dt: new Date(currentData.dt * 1000),
                    icon: currentData.weather[0].icon,
                },
                forecasts: Object.entries(
                    forecastData.list
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
                ).map(([date, forecasts]) => {
                    return {
                        date,
                        //get the min and max temp for the day
                        temp_min: Math.min(...forecasts.map(forecast => parseFloat(forecast.main.temp_min.toPrecision(2)))),
                        temp_max: Math.max(...forecasts.map(forecast => parseFloat(forecast.main.temp_max.toPrecision(2)))),
                        //get the 4th weather icon for the day => (3h * 4 ) = 12h (noon) (show the weather at noon, usually the warmest time of the day and the most relevant)
                        icon: forecasts[4].weather[0].icon,
                        dt: new Date(forecasts[0].dt * 1000),
                    }
                })

            });
            //set loading state to false
            loading = false;

            console.log('Weather data fetched successfully');
            console.log('Current weather data:', currentData);
            console.log('Forecast weather data:', forecastData);

            console.log('Current weather data:', current);
            console.log('Forecasts weather data:', forecasts);

        } catch
            (error) {
            console.error('Error fetching weather data:', error);
        }
    }

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
    const current_weather_options = {weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'};
    // - forecast weather
    const forcast_weather_options = {weekday: 'short'};

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
        {:else if current}
            <div class="current-weather">
                <h2>{current.dt.toLocaleDateString('fr-FR', current_weather_options)}</h2>
                <h1>{city.name}</h1>
                <span class="main-temp">{current.temp_max}°</span>
                <span class="weather-state">{current.weather}</span>
                <div class="temps">
                    <i class="high-temp"></i>
                    <p>{current.temp_max}°</p>
                    <span></span>
                    <i class="low-temp"></i>
                    <p>{current.temp_min}°</p>
                </div>
            </div>
            <div class="card">
                <div class="title">
                    <svg class="calendar"></svg>
                    <span>prévisions sur 5 jours</span>
                </div>
                <div class="forecast">
                    <div class="forecast-day">
                        <p>Auj.</p>
                    </div>
                    <div class="forecast-weather">
                        <p>Maint.</p>
                        <p><img class="weather-icon" src="images/{current.icon}.svg" alt=""></p>
                    </div>
                    <div class="forecast-temps">
                        <p>{current.temp_min}°</p>
                        <span></span>
                        <p>{current.temp_max}°</p>
                    </div>
                </div>
                {#each forecasts as forecast}
                    <div class="forecast">
                        <div class="forecast-day">
                            <p>{forecast.dt.toLocaleDateString('fr-FR', forcast_weather_options)}</p>
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
