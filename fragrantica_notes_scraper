import json
import logging
import time
from selenium.common.exceptions import TimeoutException
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import undetected_chromedriver as uc

# Setup logging
logging.basicConfig(
    filename="process_notes.log",
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
)

# Constants for XPaths
XPATHS = {
    "note_key_element": "/html/body/div[2]/div[2]/div[4]/div[1]/div[1]/div",
    "note_alt_name": "/html/body/div[2]/div[2]/div[4]/div[1]/div[1]/div/div[1]/div/div[1]/p/em",
    "note_profile": "/html/body/div[2]/div[2]/div[4]/div[1]/div[1]/div/div[2]/p",
    "note_images_base": "/html/body/div[2]/div[2]/div[4]/div[1]/div[1]/div/div[1]/div/div",
}


def get_text(driver, xpath, timeout=10):
    """Extract text from a given XPath."""
    try:
        element = WebDriverWait(driver, timeout).until(
            EC.presence_of_element_located((By.XPATH, xpath))
        )
        return element.text
    except Exception as e:
        logging.warning(f"Failed to get text from {xpath}: {e}")
        return None


def get_image_sources(driver, base_xpath, indices, timeout=10):
    """Extract image sources from a list of XPaths."""
    images = []
    for index in indices:
        img_xpath = f"{base_xpath}[{index}]/img"
        try:
            img_element = WebDriverWait(driver, timeout).until(
                EC.presence_of_element_located((By.XPATH, img_xpath))
            )
            images.append(img_element.get_attribute("src"))
        except Exception as e:
            logging.warning(f"Failed to get image source from {img_xpath}: {e}")
            images.append(None)
    return images


def extract_note_data(
    driver, note_link, retries=3, delay=2, timeout=10, max_page_time=30
):
    """Extract data from a note link with retry and driver restart logic."""
    for attempt in range(retries):
        try:
            driver.get(note_link)
            WebDriverWait(driver, max_page_time).until(
                EC.presence_of_element_located(
                    (By.XPATH, XPATHS["note_key_element"])
                )
            )
            note_images = get_image_sources(
                driver,
                XPATHS["note_images_base"],
                indices=[2, 3, 4],
                timeout=timeout,
            )
            return {
                "note_link": note_link,
                "note_alt_name": get_text(
                    driver, XPATHS["note_alt_name"], timeout=timeout
                ),
                "note_profile": get_text(
                    driver, XPATHS["note_profile"], timeout=timeout
                ),
                "note_image_1": note_images[0],
                "note_image_2": note_images[1],
                "note_image_3": note_images[2],
            }
        except TimeoutException:
            logging.warning(
                f"Page load for {note_link} exceeded the time limit. Attempt {attempt + 1} of {retries}."
            )
            time.sleep(delay * (2**attempt))  # Exponential backoff
        except Exception as e:
            logging.warning(
                f"Error retrieving data from {note_link}. Attempt {attempt + 1} of {retries}. Error: {e}"
            )

    logging.error(
        f"Failed to retrieve data from {note_link} after {retries} attempts. Restarting driver."
    )
    driver.quit()
    driver = initialize_driver()  # Restart driver after repeated failure
    return extract_note_data(
        driver, note_link, retries, delay, timeout, max_page_time
    )


def initialize_driver():
    """Initialize and return an undetected Chrome driver."""
    options = uc.ChromeOptions()
    options.headless = False
    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--incognito")
    return uc.Chrome(options=options)


def process_notes(data, driver, group_name):
    """Process notes sequentially and update the JSON data."""
    groups = [
        group
        for group in data["data"]
        if group["note_group_name"] == group_name
    ]

    if not groups:
        logging.info(f"No group named '{group_name}' found in the data.")
        return data

    total_notes = sum(len(group["notes"]) for group in groups)
    logging.info(
        f"Total number of notes to process in '{group_name}': {total_notes}"
    )

    for group in groups:
        for note in group["notes"]:
            note_link = note["note_link"]
            logging.info(f"Processing note link: {note_link}")

            extracted_data = extract_note_data(driver, note_link)

            if extracted_data:
                if note["note_link"] == extracted_data["note_link"]:
                    note.update(extracted_data)
                    logging.info(f"Successfully processed note: {note}")
                else:
                    logging.warning(
                        f"Mismatch between expected note link {note['note_link']} and extracted note link {extracted_data['note_link']}"
                    )
            else:
                logging.warning(
                    f"Failed to extract data for note link: {note['note_link']}"
                )

    logging.info(f"Final results of processed notes in '{group_name}':")
    for group in groups:
        for note in group["notes"]:
            logging.info(f"Final note data: {note}")

    return data


def save_enriched_data(data, input_file="./notes_output.json"):
    """Save the final enriched JSON data directly to the original file."""
    with open(input_file, "w") as f:
        json.dump(data, f, indent=4)
    print(f"Data enrichment complete! Final file updated as {input_file}")


def main():
    note_groups = [
        "CITRUS SMELLS",
        "FRUITS, VEGETABLES AND NUTS",
        "FLOWERS",
        "WHITE FLOWERS",
        "GREENS, HERBS AND FOUGERES",
        "SPICES",
        "SWEETS AND GOURMAND SMELLS",
        "WOODS AND MOSSES",
        "RESINS AND BALSAMS",
        "MUSK, AMBER, ANIMALIC SMELLS",
        "BEVERAGES",
        "NATURAL AND SYNTHETIC, POPULAR AND WEIRD",
    ]

    input_file = "./notes_output.json"
    with open(input_file, "r") as f:
        data = json.load(f)

    for group_name in note_groups:
        logging.info(f"Processing group: {group_name}")

        # Initialize a new driver for each group
        driver = initialize_driver()

        enriched_data = process_notes(data, driver, group_name)
        save_enriched_data(enriched_data, input_file)

        logging.info(
            "Waiting for 30 seconds before proceeding to the next group..."
        )
        driver.quit()
        time.sleep(30)


if __name__ == "__main__":
    main()
