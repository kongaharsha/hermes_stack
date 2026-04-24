# Project: United Online Check-in Automation

## TODOS
- [ ] Implement robust KTN/TSA PreCheck validation logic in `united_driver.js`
- [ ] Add explicit baggage selection (skip for "no checked bags")
- [ ] Integrate loyalty-info-tracker (fetch MileagePlus from `~/brain/concepts/loyalty-infos.md`)
- [ ] Implement session-persistence using `user-data-dir` to prevent WAF triggers
- [ ] Add error handling for PNR retrieval issues (currently fragile)
- [ ] Refactor driver to read credentials from environment instead of hardcoding
- [ ] Add visual verification step for boarding pass generation
