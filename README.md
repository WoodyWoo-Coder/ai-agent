import tejapi
tejapi.ApiConfig.api_key = "1gIZtLbbSSRNrYCIU3SaIOGBYemtkK"
info = tejapi.ApiConfig.info()
import datetime
import requests
import os
from zoneinfo import ZoneInfo
from google.adk.agents import LlmAgent

def get_stock_overview() -> dict:
    """Retrieves the latest changes from the stock market.
    Args:
       None.
    Returns:
        dict: status and result or error msg.
    """

    
    url = f"https://openapi.twse.com.tw/v1/exchangeReport/STOCK_DAY_ALL"
    try:
        response = requests.get(url)
        response.raise_for_status()  # Raise an error for bad responses
        data = response.json()
        # Create a list of dicts with 'Name' and 'TradeVolume', converting TradeVolume to int
        stock_list = [
            {
                "Name": item["Name"],
                "Code": item["Code"],
                "TradeVolume": int(item["TradeVolume"].replace(',', '')) if isinstance(item["TradeVolume"], str) else item["TradeVolume"]
            }
            for item in data
            if "Name" in item and "TradeVolume" in item and item["TradeVolume"].isdigit()
        ]
        # Sort by TradeVolume descending
        stock_list.sort(key=lambda x: x["TradeVolume"], reverse=True)
        # Get top 5
        top_five = stock_list[:5]
        # Prepare a readable report
        report = "前五名成交量公司:\n" + "\n".join(
            [f"{i+1}. {item['Name']} - {item['TradeVolume']}, 代碼為{item['Code']}" for i, item in enumerate(top_five)]
        )
        report += f"\n\n最新報告日期: {data[0]['Date']}" if data else "無可用數據"
        return {"status": "success", "report": report}
    except requests.exceptions.RequestException as e:
        return {
            "status": "error",
            "error_message": f"An error occurred while fetching the weather data: {str(e)}",
        }


def get_current_time(city: str) -> dict:
    """Returns the current time in a specified city.
    Args:
        city (str): The name of the city for which to retrieve the current time.
    Returns:
        dict: status and result or error msg.
    """

    if city.lower() == "new york":
        tz_identifier = "America/New_York"
    elif city.lower() == "taipei":
        tz_identifier = "Asia/Taipei"
    else:
        return {
            "status": "error",
            "error_message": (
                f"Sorry, I don't have timezone information for {city}."
            ),
        }

    tz = ZoneInfo(tz_identifier)
    now = datetime.datetime.now(tz)
    report = (
        f'The current time in {city} is {now.strftime("%Y-%m-%d %H:%M:%S %Z%z")}'
    )
    return {"status": "success", "report": report}


root_agent = LlmAgent(
    name="weather_time_agent",
    model="gemini-2.0-flash",
    description=(
        "Agent to answer questions about the stock market and weather in a city."
    ),
    instruction=(
        "You are a helpful agent who can answer user questions about the stock market and weather in a city."
    ),
    tools=[get_stock_overview, get_current_time],
)
