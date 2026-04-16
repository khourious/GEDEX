# GEDEX: GISAID EPI_SET Data Contributors Extractor

GEDEX automates the retrieval of `Virus name`, `Collection date`, `Originating laboratory`, `Submitting laboratory`, and `Authors` from GISAID EPI_SET acknowledgment pop-ups.

## Usage

### gedex_fetch.js

1. Access the EPI_SET URL and ensure the sequence blocks are expanded.

2. Open the browser's Developer Tools:

| OS            | Shortcut               |
| ------------- | ---------------------- |
| macOS         | `Option + Command + I` |
| Linux/Windows | `Control + Shift + I`  |

3. Copy the script below, paste it into the console, and press `Enter`.

<details>
<summary>gedex_fetch.js</summary>


```javascript
// gedex_fetch.js

(function() {
	'use strict';

	const config = {
		delay: 3000,
		scrollDelay: 600,
		batchSize: 10,
		retryAttempts: 5
	};

	const allData = [];
	let failedItems = [];
	const elements = Array.from(document.querySelectorAll('.accid, [call_id]'))
		.filter(el => {
			const callId = el.getAttribute('call_id');
			return callId && callId.includes('EPI_');
		});

	console.log(`${elements.length} elements found`);

	const panel = document.createElement('div');
	panel.style.cssText = `
		position: fixed;
		bottom: 20px;
		right: 20px;
		background: #004d43;
		color: white;
		padding: 15px;
		border-radius: 10px;
		z-index: 100000;
		font-family: Arial;
		box-shadow: 0 4px 20px rgba(0,0,0,0.4);
		min-width: 150px;
	`;

	const createUI = () => {
		panel.innerHTML = '';

		const title = document.createElement('div');
		title.textContent = 'GEDEX';
		title.style.fontWeight = 'bold';
		title.style.marginBottom = '10px';

		const progressDiv = document.createElement('div');
    progressDiv.style.marginBottom = '10px';
		progressDiv.innerHTML = `
			<div>Progress: <span id="counter">0</span>/${elements.length}</div>
			<div>Collected: <span id="collected">0</span></div>
		`;

		const startBtn = document.createElement('button');
		startBtn.id = 'startBtn';
		startBtn.textContent = 'Run';
		startBtn.style.cssText = 'width: 100%; padding: 10px; background: #00b894; color: white; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; font-size: 12px;';

		panel.appendChild(title);
		panel.appendChild(progressDiv);
		panel.appendChild(startBtn);
	};

	createUI();

	document.body.appendChild(panel);

	let isRunning = false;
	let currentIndex = 0;

	function updateDisplay() {
		document.getElementById('counter').textContent = currentIndex;
		document.getElementById('collected').textContent = allData.length;
	}

	async function extractElementData(element, attempt = 0) {
		const callId = element.getAttribute('call_id');
		const epiId = callId.split('/')[0];

		console.log(`Processing ${currentIndex + 1}/${elements.length}: ${epiId}`);

		element.scrollIntoView({ behavior: 'smooth', block: 'center' });

		await new Promise(resolve => setTimeout(resolve, config.scrollDelay));

		const initialScrollY = window.scrollY;

		element.click();

		const mouseoverEvent = new MouseEvent('mouseover', {
			view: window,
			bubbles: true,
			cancelable: true
		});

		element.dispatchEvent(mouseoverEvent);

		await new Promise(resolve => setTimeout(resolve, config.delay));

		let popupContent = null;

		const iframes = Array.from(document.querySelectorAll('iframe'));
		for (const iframe of iframes) {
			try {
				const style = window.getComputedStyle(iframe);
				if (style.display !== 'none' && style.visibility !== 'hidden') {
					const doc = iframe.contentDocument || iframe.contentWindow.document;
					if (doc && doc.body) {
						const text = doc.body.innerText;
						if (text && text.includes('EPI_')) {
							popupContent = text;
							console.log(`Popup found via iframe for ${epiId}`);
							break;
						}
					}
				}
			} catch (e) {
			}
		}

		if (!popupContent && attempt < config.retryAttempts) {
			console.log(`Attempt ${attempt + 1} failed, trying again...`);

			window.scrollTo(0, initialScrollY);

			const mouseoutEvent = new MouseEvent('mouseout', {
				view: window,
				bubbles: true,
				cancelable: true
			});

			element.dispatchEvent(mouseoutEvent);

			await new Promise(resolve => setTimeout(resolve, 500));

			return extractElementData(element, attempt + 1);
		}

		const mouseoutEvent = new MouseEvent('mouseout', {
			view: window,
			bubbles: true,
			cancelable: true
		});

		element.dispatchEvent(mouseoutEvent);

		if (popupContent) {
			allData.push({
				callId: callId,
				content: popupContent,
				timestamp: new Date().toISOString(),
				elementText: element.innerText.trim(),
				attempts: attempt + 1
			});

			console.log(`Success: ${epiId} (attempts: ${attempt + 1})`);
			return true;
		} else {
			console.log(`Failed: ${epiId} after ${attempt + 1} attempts`);
			return false;
		}
	}

	async function retryFailedItems() {
		if (failedItems.length === 0) return;
		
		console.log(`Retrying ${failedItems.length} failed items...`);
		
		const itemsToRetry = [...failedItems];
		failedItems = [];
		
		for (let i = 0; i < itemsToRetry.length && isRunning; i++) {
			const item = itemsToRetry[i];
			const element = item.element;
			const originalIndex = item.index;
			
			console.log(`Retrying failed item ${i + 1}/${itemsToRetry.length}: ${element.getAttribute('call_id').split('/')[0]}`);
			
			const success = await extractElementData(element);
			
			if (!success) {
				failedItems.push(item);
				console.log(`Still failed: ${element.getAttribute('call_id').split('/')[0]}`);
			}
			
			if (i < itemsToRetry.length - 1) {
				await new Promise(resolve => setTimeout(resolve, 500));
			}
		}
		
		if (failedItems.length > 0) {
			console.log(`${failedItems.length} items still failed after retry`);
		}
	}

	async function processBatch() {
		if (!isRunning || currentIndex >= elements.length) {
			if (currentIndex >= elements.length) {
				await retryFailedItems();
				
				document.getElementById('startBtn').disabled = false;
				document.getElementById('startBtn').textContent = 'COMPLETED';
				exportData();
			}
			return;
		}

		const batchEnd = Math.min(currentIndex + config.batchSize, elements.length);

		for (let i = currentIndex; i < batchEnd && isRunning; i++) {
			
			const success = await extractElementData(elements[i]);

			if (!success) {
				failedItems.push({
					element: elements[i],
					index: i
				});
				console.log(`Added to retry list: ${elements[i].getAttribute('call_id').split('/')[0]}`);
			}

			currentIndex = i + 1;
			updateDisplay();

			if (i < batchEnd - 1) {
				await new Promise(resolve => setTimeout(resolve, 300));
			}
		}

		if (isRunning && currentIndex < elements.length) {
			setTimeout(processBatch, 500);
		} else if (currentIndex >= elements.length) {
			await retryFailedItems();
			
			document.getElementById('startBtn').disabled = false;
			document.getElementById('startBtn').textContent = 'COMPLETED';
			exportData();
		}
	}

	function exportData() {
		if (allData.length === 0) {
			alert('No data to export.');
			return;
		}

		const json = JSON.stringify(allData, null, 2);
		const blob = new Blob([json], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `gisaid_epi_set_data_contributors_${new Date().toISOString().slice(0,10)}.json`;
		a.click();

		console.log(`Exported: ${allData.length} records`);

		console.group('EXTRACTION SUMMARY');
		console.log(`Total elements: ${elements.length}`);
		console.log(`Successfully collected: ${allData.length}`);
		console.log(`Success rate: ${((allData.length/elements.length)*100).toFixed(1)}%`);
		console.groupEnd();
	}

	document.getElementById('startBtn').addEventListener('click', () => {
		if (!isRunning) {
			isRunning = true;
			document.getElementById('startBtn').disabled = true;
			document.getElementById('startBtn').textContent = 'PROCESSING...';
			processBatch();
		}
	});

	updateDisplay();

	window.gisaidData = allData;
	window.startExtraction = () => document.getElementById('startBtn').click();
	window.exportGisaidData = exportData;

	console.log('Extractor loaded!');
	console.log('Available commands:');
	console.log('- startExtraction() to start');
	console.log('- exportGisaidData() to export');
	console.log('- gisaidData to access data');
})();
```

</details>

4. Click `Run` in the GEDEX panel. Upon completion, a JSON file is automatically downloaded.

### gedex_parse.ipynb

1. Go to the [Google Colab Platform](https://colab.research.google.com)

2. Under `Open notebook`, select the `Github`

3. Enter the following link in `Enter a GitHub URL or search by organisation or user`:

```
https://github.com/khourious/GEDEX
```

4. Click `search`, and after a brief scan, select `gedex_parse.ipynb`.

5. Upload the JSON file obtained from `gedex_fetch.js` using the file browser panel.

6. Click `Run all` in the Colab toolbar. Upon completion, a CSV file is automatically downloaded.

## Notes

- The `gedex_fetch.js` sequentially clicks through each visible EPI_ISL entry in the expanded EPI_SET view, waits for the acknowledgment pop-up to render, and captures its content.
- Due to variable page loading and popup behavior, some entries may fail initially. The `gedex_fetch.js` retries each popup up to three times and queues failures for a second pass. A small subset may remain inaccessible depending on browser or rendering conditions.
- The `gedex_parse.ipynb` parses the raw JSON content, extracts structured metadata fields, removes duplicate accessions, and exports a CSV file.