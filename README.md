// Open-Meteo API - Free, no API key required!
const WEATHER_API_BASE = 'https://api.open-meteo.com/v1';
const GEOCODING_API = 'https://geocoding-api.open-meteo.com/v1';

// DOM Elements
const cityInput = document.getElementById('cityInput');
const searchBtn = document.getElementById('searchBtn');
const locationBtn = document.getElementById('locationBtn');
const weatherContainer = document.getElementById('weatherContainer');
const loading = document.getElementById('loading');
const error = document.getElementById('error');
const errorMessage = document.getElementById('errorMessage');

// Event Listeners
searchBtn.addEventListener('click', () => searchWeather());
cityInput.addEventListener('keypress', (e) => {
    if (e.key === 'Enter') searchWeather();
});
locationBtn.addEventListener('click', getUserLocation);

// Search weather by city name
async function searchWeather() {
    const city = cityInput.value.trim();
    if (!city) {
        showError('Please enter a city name');
        return;
    }
    
    await getWeatherByCity(city);
}

// Get weather by city name
async function getWeatherByCity(city) {
    showLoading();
    
    try {
        // First, get coordinates from city name
        const geoResponse = await fetch(
            `${GEOCODING_API}/search?name=${encodeURIComponent(city)}&count=1&language=en&format=json`
        );
        
        if (!geoResponse.ok) {
            throw new Error('City not found');
        }
        
        const geoData = await geoResponse.json();
        
        if (!geoData.results || geoData.results.length === 0) {
            throw new Error('City not found. Please try again.');
        }
        
        const location = geoData.results[0];
        const { latitude, longitude, name, country } = location;
        
        // Now get weather data
        await fetchWeatherData(latitude, longitude, `${name}, ${country}`);
        
    } catch (err) {
        showError(err.message);
    } finally {
        hideLoading();
    }
}

// Get user's current location
function getUserLocation() {
    if (navigator.geolocation) {
        showLoading();
        navigator.geolocation.getCurrentPosition(
            async (position) => {
                const { latitude, longitude } = position.coords;
                try {
                    // Get city name from coordinates
                    const response = await fetch(
                        `https://api.bigdatacloud.net/v1/reverse-geo?latitude=${latitude}&longitude=${longitude}&localityLanguage=en`
                    );
                    const data = await response.json();
                    const cityName = data.city || data.locality || 'Your Location';
                    
                    await fetchWeatherData(latitude, longitude, cityName);
                } catch (err) {
                    await fetchWeatherData(latitude, longitude, 'Your Location');
                } finally {
                    hideLoading();
                }
            },
            (error) => {
                hideLoading();
                showError('Unable to retrieve your location. Please enter city name manually.');
            }
        );
    } else {
        showError('Geolocation is not supported by your browser.');
    }
}

// Fetch weather data from Open-Meteo API
async function fetchWeatherData(lat, lon, locationName) {
    try {
        const weatherUrl = `${WEATHER_API_BASE}/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,is_day,precipitation,weather_code,cloud_cover,pressure_msl,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min,uv_index_max&timezone=auto`;
        
        const response = await fetch(weatherUrl);
        
        if (!response.ok) {
            throw new Error('Failed to fetch weather data');
        }
        
        const data = await response.json();
        displayWeather(data, locationName);
        
    } catch (err) {
        showError(err.message);
    }
}

// Display weather data
function displayWeather(data, locationName) {
    const current = data.current;
    const daily = data.daily;
    
    // Update main weather info
    document.getElementById('cityName').textContent = locationName;
    document.getElementById('date').textContent = new Date().toLocaleDateString('en-US', {
        weekday: 'long',
        year: 'numeric',
        month: 'long',
        day: 'numeric'
    });
    
    // Get weather icon and description
    const weatherInfo = getWeatherDescription(current.weather_code);
    document.getElementById('weatherIcon').src = weatherInfo.icon;
    document.getElementById('description').textContent = weatherInfo.description;
    
    // Temperature
    document.getElementById('temp').textContent = `${Math.round(current.temperature_2m)}°C`;
    
    // Weather details
    document.getElementById('windSpeed').textContent = `${current.wind_speed_10m} km/h`;
    document.getElementById('humidity').textContent = `${current.relative_humidity_2m}%`;
    document.getElementById('pressure').textContent = `${current.pressure_msl} hPa`;
    document.getElementById('visibility').textContent = '10 km'; // Approximate
    document.getElementById('feelsLike').textContent = `${Math.round(current.apparent_temperature)}°C`;
    document.getElementById('uvIndex').textContent = daily.uv_index_max[0];
    
    // Display forecast
    displayForecast(daily);
    
    // Show weather container
    weatherContainer.style.display = 'block';
    error.style.display = 'none';
}

// Display 5-day forecast
function displayForecast(daily) {
    const forecastContainer = document.getElementById('forecastContainer');
    forecastContainer.innerHTML = '';
    
    for (let i = 1; i <= 5; i++) {
        const date = new Date(daily.time[i]);
        const dayName = date.toLocaleDateString('en-US', { weekday: 'short' });
        const weatherCode = daily.weather_code[i];
        const maxTemp = Math.round(daily.temperature_2m_max[i]);
        const minTemp = Math.round(daily.temperature_2m_min[i]);
        
        const weatherInfo = getWeatherDescription(weatherCode);
        
        const forecastCard = document.createElement('div');
        forecastCard.className = 'forecast-card';
        forecastCard.innerHTML = `
            <div class="day">${dayName}</div>
            <div class="forecast-icon">
                <img src="${weatherInfo.icon}" alt="${weatherInfo.description}">
            </div>
            <div class="temp-range">
                <span class="max">${maxTemp}°</span> / <span class="min">${minTemp}°</span>
            </div>
        `;
        
        forecastContainer.appendChild(forecastCard);
    }
}

// Get weather description and icon based on WMO code
function getWeatherDescription(code) {
    const weatherCodes = {
        0: { description: 'Clear sky', icon: 'https://openweathermap.org/img/wn/01d.png' },
        1: { description: 'Mainly clear', icon: 'https://openweathermap.org/img/wn/02d.png' },
        2: { description: 'Partly cloudy', icon: 'https://openweathermap.org/img/wn/02d.png' },
        3: { description: 'Overcast', icon: 'https://openweathermap.org/img/wn/03d.png' },
        45: { description: 'Foggy', icon: 'https://openweathermap.org/img/wn/50d.png' },
        48: { description: 'Depositing rime fog', icon: 'https://openweathermap.org/img/wn/50d.png' },
        51: { description: 'Light drizzle', icon: 'https://openweathermap.org/img/wn/09d.png' },
        53: { description: 'Moderate drizzle', icon: 'https://openweathermap.org/img/wn/09d.png' },
        55: { description: 'Dense drizzle', icon: 'https://openweathermap.org/img/wn/09d.png' },
        61: { description: 'Slight rain', icon: 'https://openweathermap.org/img/wn/10d.png' },
        63: { description: 'Moderate rain', icon: 'https://openweathermap.org/img/wn/10d.png' },
        65: { description: 'Heavy rain', icon: 'https://openweathermap.org/img/wn/10d.png' },
        71: { description: 'Slight snow', icon: 'https://openweathermap.org/img/wn/13d.png' },
        73: { description: 'Moderate snow', icon: 'https://openweathermap.org/img/wn/13d.png' },
        75: { description: 'Heavy snow', icon: 'https://openweathermap.org/img/wn/13d.png' },
        77: { description: 'Snow grains', icon: 'https://openweathermap.org/img/wn/13d.png' },
        80: { description: 'Slight rain showers', icon: 'https://openweathermap.org/img/wn/09d.png' },
        81: { description: 'Moderate rain showers', icon: 'https://openweathermap.org/img/wn/09d.png' },
        82: { description: 'Violent rain showers', icon: 'https://openweathermap.org/img/wn/09d.png' },
        85: { description: 'Slight snow showers', icon: 'https://openweathermap.org/img/wn/13d.png' },
        86: { description: 'Heavy snow showers', icon: 'https://openweathermap.org/img/wn/13d.png' },
        95: { description: 'Thunderstorm', icon: 'https://openweathermap.org/img/wn/11d.png' },
        96: { description: 'Thunderstorm with hail', icon: 'https://openweathermap.org/img/wn/11d.png' },
        99: { description: 'Thunderstorm with heavy hail', icon: 'https://openweathermap.org/img/wn/11d.png' }
    };
    
    return weatherCodes[code] || { description: 'Unknown', icon: 'https://openweathermap.org/img/wn/01d.png' };
}

// Utility functions
function showLoading() {
    loading.style.display = 'block';
    weatherContainer.style.display = 'none';
    error.style.display = 'none';
}

function hideLoading() {
    loading.style.display = 'none';
}

function showError(message) {
    errorMessage.textContent = message;
    error.style.display = 'block';
    weatherContainer.style.display = 'none';
    hideLoading();
}

// Initialize with a default city (optional)
window.addEventListener('load', () => {
    // You can uncomment this to load a default city
    // getWeatherByCity('London');
});
