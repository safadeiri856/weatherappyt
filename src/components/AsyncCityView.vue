<template>
  <div class="flex flex-col flex-1 items-center">
    <!-- Banner -->
    <div
      v-if="route.query.preview"
      class="text-white p-4 bg-weather-secondary w-full text-center"
    >
      <p>
        You are currently previewing this city, click the "+"
        icon to start tracking this city.
      </p>
    </div>
    <!-- Weather Overview -->
    <div class="flex flex-col items-center text-white py-12">
      <h1 class="text-4xl mb-2">{{ route.params.city }}</h1>
      <p class="text-sm mb-12">
        {{
          new Date(weatherData.currentTime).toLocaleDateString(
            "en-us",
            {
              weekday: "short",
              day: "2-digit",
              month: "long",
            }
          )
        }}
        {{
          new Date(weatherData.currentTime).toLocaleTimeString(
            "en-us",
            {
              timeStyle: "short",
            }
          )
        }}
      </p>
      <p class="text-8xl mb-8">
        {{ Math.round(weatherData.current.temp) }}&deg;
      </p>
      <p>
        Feels like
        {{ Math.round(weatherData.current.feels_like) }} &deg;
      </p>
      <p class="capitalize">
        {{ weatherData.current.weather[0].description }}
      </p>
      <img
        class="w-[150px] h-auto"
        :src="
          `http://openweathermap.org/img/wn/${weatherData.current.weather[0].icon}@2x.png`
        "
        alt=""
      />
    </div>

    <hr class="border-white border-opacity-10 border w-full" />

    <!-- Hourly Weather -->
    <div class="max-w-screen-md w-full py-12">
      <div class="mx-8 text-white">
        <h2 class="mb-4">Hourly Weather</h2>
        <div class="flex gap-10 overflow-x-scroll">
          <div
            v-for="hourData in weatherData.hourly"
            :key="hourData.dt"
            class="flex flex-col gap-4 items-center"
          >
            <p class="whitespace-nowrap text-md">
              {{
                new Date(
                  hourData.currentTime
                ).toLocaleTimeString("en-us", {
                  hour: "numeric",
                })
              }}
            </p>
            <img
              class="w-auto h-[50px] object-cover"
              :src="
                `http://openweathermap.org/img/wn/${hourData.weather[0].icon}@2x.png`
              "
              alt=""
            />
            <p class="text-xl">
              {{ Math.round(hourData.temp) }}&deg;
            </p>
          </div>
        </div>
      </div>
    </div>

    <hr class="border-white border-opacity-10 border w-full" />

    <!-- Weekly Weather -->
    <div class="max-w-screen-md w-full py-12">
      <div class="mx-8 text-white">
        <h2 class="mb-4">7 Day Forecast</h2>
        <div
          v-for="day in weatherData.daily"
          :key="day.dt"
          class="flex items-center"
        >
          <p class="flex-1">
            {{
              new Date(day.dt * 1000).toLocaleDateString(
                "en-us",
                {
                  weekday: "long",
                }
              )
            }}
          </p>
          <img
            class="w-[50px] h-[50px] object-cover"
            :src="
              `http://openweathermap.org/img/wn/${day.weather[0].icon}@2x.png`
            "
            alt=""
          />
          <div class="flex gap-2 flex-1 justify-end">
            <p>H: {{ Math.round(day.temp.max) }}</p>
            <p>L: {{ Math.round(day.temp.min) }}</p>
          </div>
        </div>
      </div>
    </div>

    <div
      class="flex items-center gap-2 py-12 text-white cursor-pointer duration-150 hover:text-red-500"
      @click="removeCity"
    >
      <i class="fa-solid fa-trash"></i>
      <p>Remove City</p>
    </div>
  </div>
</template>

<script setup>
import axios from "axios";
import { useRoute, useRouter } from "vue-router";

const route = useRoute();
const apiKey = "8908f0df0cf681d68d6ba2fa7c141572";

const getWeatherData = async () => {
  try {
    const lat = route.query.lat;
    const lon = route.query.lng;

    // 1) الطقس الحالي
    const currentRes = await axios.get(
      `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey}&units=imperial`
    );

    // 2) التوقعات (كل 3 ساعات لمدة 5 أيام)
    const forecastRes = await axios.get(
      `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&appid=${apiKey}&units=imperial`
    );

    const localOffset = new Date().getTimezoneOffset() * 60000;
    const timezoneOffset = currentRes.data.timezone * 1000; // بالثواني -> ملي ثانية

    // تجهيز current
    const current = {
      temp: currentRes.data.main.temp,
      feels_like: currentRes.data.main.feels_like,
      weather: currentRes.data.weather,
    };

    const utcNow = Date.now() + localOffset;
    const currentTime = utcNow + timezoneOffset;

    // تجهيز hourly (من forecast مباشرة - كل عنصر كل 3 ساعات)
    const hourly = forecastRes.data.list.map((item) => {
      const utc = item.dt * 1000 + localOffset;
      return {
        dt: item.dt,
        currentTime: utc + timezoneOffset,
        weather: item.weather,
        temp: item.main.temp,
      };
    });

    // تجهيز daily (نجمع بيانات كل يوم من الـ forecast يدويًا)
    const dailyMap = {};
    forecastRes.data.list.forEach((item) => {
      const dateKey = new Date(item.dt * 1000).toLocaleDateString("en-us");
      if (!dailyMap[dateKey]) {
        dailyMap[dateKey] = {
          dt: item.dt,
          weather: item.weather,
          temp: { max: item.main.temp_max, min: item.main.temp_min },
        };
      } else {
        dailyMap[dateKey].temp.max = Math.max(
          dailyMap[dateKey].temp.max,
          item.main.temp_max
        );
        dailyMap[dateKey].temp.min = Math.min(
          dailyMap[dateKey].temp.min,
          item.main.temp_min
        );
      }
    });
    const daily = Object.values(dailyMap);
await new Promise((res) => setTimeout(res, 1000));
    return {
      currentTime,
      current,
      hourly,
      daily,
    };
    
  } catch (err) {
    console.log(err);
  }
};

const weatherData = await getWeatherData();

const router = useRouter();
const removeCity = () => {
  const savedData = localStorage.getItem("savedCities");

  if (!savedData) {
    router.push({
      name: "home",
    });
    return;
  }

  const cities = JSON.parse(savedData);
  const updatedCities = cities.filter(
    (city) => city.id !== route.query.id
  );
  localStorage.setItem(
    "savedCities",
    JSON.stringify(updatedCities)
  );
  router.push({
    name: "home",
  });
};
</script>