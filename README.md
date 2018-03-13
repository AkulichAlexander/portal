# portal
example code for search hotel logic

function HotelsViewModel() {
    var self = this,
        map = new google.maps.Map(document.getElementById('mainHotelMap'), { zoom: 14 }),
        infoWindow = new google.maps.InfoWindow(),
        markerIcon = {
            url: '/Views/CustomViews/4/style/img/map-icon-hotel.png',
            origin: new google.maps.Point(0, -8)
        },
        activeMarkerIcon = {
            url: '/Views/CustomViews/4/style/img/map-icon-hotel-active.png',
            origin: new google.maps.Point(0, -8)
        },
        PAGE_SIZE = 10,
        allHotels = [],
        filteredHotels = [],
        pageNumber = 0,
        $container = $('#_Aside-filter-hotel'),
        $filters = $container.find('.side-toggle'),
        $window = $(window);


    function scrollToTheTop() {
        $('body,html').animate({ scrollTop: $container.offset().top - 100 }, 300);
    }

    function showFirstPage() {
        infoWindow && infoWindow.close();
        self.hotels(filteredHotels.slice(0, PAGE_SIZE));
        pageNumber = 1;
    }

    function applySorting(sorter) {
        allHotels.sort(sorter);
        filteredHotels.sort(sorter);
        showFirstPage();
    }

    function trustYouScoreToRate(score) {
        return score ? Math.round((score * 50.0) / 100.0) / 10.0 : '';
    }

    $window.on('scroll',
        function () {
            if (!$container.is(':visible')) return;

            var distance = $('.footer-top').offset().top - ($window.scrollTop() + $window.height());
            if (distance >= 300) return;

            var count = pageNumber * PAGE_SIZE;
            if (count >= filteredHotels.length) return;

            pageNumber++;
            var nextPage = filteredHotels.slice(count, count + PAGE_SIZE);
            ko.utils.arrayPushAll(self.hotels, nextPage);
        });

    self.hotels = ko.observableArray([]).extend({ deferred: true });
    self.isLoading = ko.observable(false);
    self.isCitySearch = ko.observable(false);
    self.total = ko.observable(0);
    self.filtered = ko.observable(0);

    self.showFilterReset = ko.pureComputed(function () {
        return this.filtered() < this.total();
    }, self);

    self.find = function (__, e) {
        var params = self.searchBox;

        if (!params.itemId && !params.itemType) {
            params.valid(false);
        }

        if (!params.valid()) {
            e && e.stopPropagation();
            $('#search-hotel').focus();
            return;
        }

        var type = params.itemType,
            checkIn = moment(params.checkIn()).format('YYYY.MM.DD'),
            checkOut = moment(params.checkOut()).format('YYYY.MM.DD');

        if (type === 'VA') {
            var source = 'va';

            var url = '/Hotel/' + source + '-' + params.itemId + '-' + params.term() + '?'
                + 'CheckIn=' + checkIn
                + '&CheckOut=' + checkOut
                + '&Adult=' + params.adults()
                + '&HotelChild=' + params.children();

            document.location.href = url;

            self.isLoading(true);
            self.hotels.removeAll();
            Home.resetSorting($('#_Aside-filter-hotel .sort-button'));

            return;
        }

        $.get('/Hotel/Find', {
            term: params.term(),
            itemType: type,
            lat: params.latitude,
            lon: params.longitude,
            regionId: params.itemId,
            landmarkId: params.landmarkId,
            categoryId: type === 'HC' ? params.itemId : '',
            categoryUrl: type === 'HC' ? params.itemUrl : '',
            checkIn: checkIn,
            checkOut: checkOut,
            adult: params.adults(),
            children: params.children(),
            childAges: params.childAgesString()
        },
            function (data) {

                allHotels = _.map(data, function (hotel) { return buildHotelViewModel(hotel); });

                var i = allHotels.length,
                    searchBox = self.searchBox;

                self.total(i);
                self.filtered(i);
                while (i--) filteredHotels[i] = allHotels[i];

                if (allHotels.length > 0 && ((!searchBox.latitude && searchBox.latitude !== 0) || (!searchBox.longitude && searchBox.longitude !== 0))) {
                    searchBox.latitude = allHotels[0].latitude;
                    searchBox.longitude = allHotels[0].longitude;
                }

                self.mapFilters.centerLatitude = searchBox.latitude;
                self.mapFilters.centerLongitude = searchBox.longitude;

                map.setCenter(new google.maps.LatLng(searchBox.latitude, searchBox.longitude));
                map.setZoom(14);
                showFirstPage();
                Home.openFilters($filters);
                self.isLoading(false);
            });

        self.isLoading(true);
        self.isCitySearch(type === 'C');
        self.mapFilters.clear();
        allHotels = [];
        filteredHotels = [];
        pageNumber = 0;
        self.total(0);
        self.filtered(0);
        self.hotels.removeAll();
        self.filters.reset();
        Home.resetSorting($('#_Aside-filter-hotel .sort-button'));
        Home.openTab('#_Aside-filter-hotel');
    };

    self.sortByPrice = function (__, e) {
        applySorting(Home.getSortDirection($(e.target))
            ? function (a, b) { return a.price() - b.price(); }
            : function (a, b) { return b.price() - a.price(); });
    };

    self.sortByStars = function (__, e) {
        applySorting(Home.getSortDirection($(e.target), true)
            ? function (a, b) { return a.stars - b.stars; }
            : function (a, b) { return b.stars - a.stars; });
    };

    self.sortByReviews = function (__, e) {
        applySorting(Home.getSortDirection($(e.target), true)
            ? function (a, b) { return a.trustYouScore - b.trustYouScore; }
            : function (a, b) { return b.trustYouScore - a.trustYouScore; });
    };

    self.sortByDistance = function (__, e) {
        applySorting(Home.getSortDirection($(e.target))
            ? function (a, b) { return (a.distanceToCenter || 100000) - (b.distanceToCenter || 100000); }
            : function (a, b) { return (b.distanceToCenter || 100000) - (a.distanceToCenter || 100000); });
    };

    self.hotels.subscribe(function (changes) {
        var t = 0, l = changes.length;

        for (var i = 0; i < l; ++i) {
            var change = changes[i];

            if (change.moved >= 0) continue;

            var status = change.status,
                hotel = change.value;

            if (status === 'added') {

                if (!hotel.marker && hotel.latitude && hotel.longitude) {
                    hotel.marker = new google.maps.Marker({
                        position: {
                            lat: hotel.latitude,
                            lng: hotel.longitude
                        },
                        map: map,
                        visible: false,
                        icon: markerIcon,
                        animation: google.maps.Animation.DROP,
                        title: hotel.title,
                        label: {
                            text: hotel.price().toString()
                        }
                    });

                    (function (h) {
                        var marker = h.marker;

                        google.maps.event.addListener(marker, 'mouseover', h.activateMarker);
                        google.maps.event.addListener(marker, 'mouseout', h.deactivateMarker);
                        google.maps.event.addListener(marker, 'click', function () {
                            h.scrollTo();
                            h.openInfoWindow();
                        });

                    })(hotel);
                }

                if (hotel.marker && !hotel.marker.getVisible()) {
                    (function (h, delay) {
                        h.markerTimeoutId = setTimeout(function () {
                            h.marker.setAnimation(google.maps.Animation.DROP);
                            h.marker.setVisible(true);
                            delete h.markerTimeoutId;
                        }, 100 + delay);
                    })(hotel, ++t * 100);
                }

            } else if (status === 'deleted') {

                if (hotel.markerTimeoutId) {
                    clearTimeout(hotel.markerTimeoutId);
                }

                if (hotel.marker && hotel.marker.getVisible()) {
                    hotel.marker.setVisible(false);
                }
            }
        }
    }, null, 'arrayChange');

    self.searchBox = new SearchBoxViewModel({
        maxChildren: 3,
        find: self.find,
        data: {
            adults: 1,
            children: 0,
            checkIn: new Date(new Date(Home.INITIAL_DATE_FROM).setDate(Home.INITIAL_DATE_FROM.getDate() + 1)),
            checkOut: new Date(new Date(Home.INITIAL_DATE_FROM).setDate(Home.INITIAL_DATE_FROM.getDate() + 2)), // one day interval
            term: '',
            valid: true
        }
    });

    self.resizeMap = function () {
        setTimeout(function () {
            google.maps.event.trigger(map, 'resize');
        }, 200);
    };

    $('#search-hotel')
        .keypress(function (e) {
            if (e.which === 13) {
                self.find(null, e);
            } else {
                self.searchBox.valid(false);
                self.searchBox.itemId = '';
                self.searchBox.itemType = '';
            }
        })
        .hotelAutoComplete({
            delay: 200,
            minLength: 2,
            source: '/Hotel/Autocomplete',
            select: function (e, ui) {
                var item = ui.item,
                    type = item.category,
                    searchBox = self.searchBox;

                searchBox.term(item.title).valid(true);
                searchBox.itemId = item.hotelId || item.cityId;
                searchBox.itemType = type === 3 ? 'VA' : 'C';

                searchBox.landmarkId = item.landmarkId || null;

                if (item.latitude && item.longitude) {
                    searchBox.latitude = item.latitude;
                    searchBox.longitude = item.longitude;
                } else {
                    searchBox.latitude = null;
                    searchBox.longitude = null;
                }
            }
        });


    function FilterBoxViewModel() {
        var filter = this,
            MIN_BUDGET = 0,
            MAX_BUDGET = 1000,
            MIN_RATING = 0,
            MAX_RATING = 100,
            MIN_DISTANCE = 0,
            MAX_DISTANCE = 10000;

        var applyFilters = function () {
            scrollToTheTop();

            var showTourGlobPickOnly = filter.showTourGlobPickOnly(),
                stars = filter.stars() * 1.0,
                from = filter.fromBudget(),
                to = filter.toBudget(),
                fromRating = filter.fromRating(),
                toRating = filter.toRating(),
                fromDistance = filter.fromDistance(),
                toDistance = filter.toDistance(),
                hotelName = (filter.hotelName() || '').trim().toLowerCase();

            filteredHotels = [];
            var len = allHotels.length, i = -1, j = -1;

            while (++i < len) {
                var hotel = allHotels[i];

                if (showTourGlobPickOnly && !hotel.tourGlobPick) {
                    continue;
                }

                if (hotelName && hotel.title.toLowerCase().indexOf(hotelName) === -1) {
                    continue;
                }

                if (stars && (hotel.stars !== stars && hotel.stars !== stars + 0.5)) {
                    continue;
                }

                if ((from > MIN_BUDGET && hotel.price() < from) || (to < MAX_BUDGET && hotel.price() > to)) {
                    continue;
                }

                if ((fromDistance > MIN_DISTANCE && hotel.distanceToCenter < fromDistance) || (toDistance < MAX_DISTANCE && hotel.distanceToCenter > toDistance)) {
                    continue;
                }

                if ((fromRating > MIN_RATING && hotel.trustYouScore < fromRating) || (toRating < MAX_RATING && hotel.trustYouScore > toRating)) {
                    continue;
                }

                filteredHotels[++j] = allHotels[i];
            }

            self.filtered(j + 1);
            showFirstPage();
        };

        var trustYouLabel = function (scoreFrom, scoreTo) {
            return loc.d_2trustyourating + ': ' + trustYouScoreToRate(scoreFrom) + ' ' + loc.d_2timeto + ' ' + trustYouScoreToRate(scoreTo);
        };

        var distanceLabel = function (distanceFrom, distanceTo) {
            return loc.d_2distancefromcenter + ': ' + (Math.round((distanceFrom / 100.0)) / 10.0) + ' ' + loc.d_2timeto + ' ' + (Math.round((distanceTo / 100.0)) / 10.0)
                + (distanceTo === MAX_DISTANCE ? '+' : '') + ' ' + loc.d_2km;
        };

        filter.hotelName = ko.observable(null);

        filter.showTourGlobPickOnly = ko.observable(false);

        filter.fromBudget = ko.observable(MIN_BUDGET);
        filter.toBudget = ko.observable(MAX_BUDGET);
        filter.budgetRange = ko.observable(MIN_BUDGET + ' ' + loc.d_2timeto + ' ' + MAX_BUDGET + '+');

        filter.stars = ko.observable(0);

        filter.fromRating = ko.observable(MIN_RATING);
        filter.toRating = ko.observable(MAX_RATING);
        filter.ratingLabel = ko.observable(trustYouLabel(MIN_RATING, MAX_RATING));

        filter.fromDistance = ko.observable(MIN_DISTANCE);
        filter.toDistance = ko.observable(MAX_DISTANCE);
        filter.distanceLabel = ko.observable(distanceLabel(MIN_DISTANCE, MAX_DISTANCE));

        filter.starsAreSet = ko.pureComputed(function () {
            return this.stars() > 0;
        }, filter);

        filter.starsLabel = ko.pureComputed(function () {
            var stars = this.stars();
            return loc.d_2starnumber + (stars > 0 ? '  (' + stars + ')' : '');
        }, filter);

        filter.clearStars = function () {
            filter.stars(0);
        };

        filter.budgetLabel = ko.pureComputed(function () {
            return loc.d_2budget + ': ' + this.budgetRange() + ' ' + PortalV3.currencyCode();
        }, filter);

        filter.reset = function () {
            filter.hotelName(null);
            filter.hotelName.notifySubscribersImmediately(filter.hotelName());

            filter.stars(0);

            filter.showTourGlobPickOnly(false);

            $('#price-range-hotel').slider({ values: [MIN_BUDGET, MAX_BUDGET] });
            filter.fromBudget(MIN_BUDGET);
            filter.toBudget(MAX_BUDGET);
            filter.budgetRange(MIN_BUDGET + ' ' + loc.d_2timeto + ' ' + MAX_BUDGET + '+');

            $('#rating-range-hotel').slider({ values: [MIN_RATING, MAX_RATING] });
            filter.fromRating(MIN_RATING);
            filter.toRating(MAX_RATING);
            filter.ratingLabel(trustYouLabel(MIN_RATING, MAX_RATING));

            $('#distance-range-hotel').slider({ values: [MIN_DISTANCE, MAX_DISTANCE] });
            filter.fromDistance(MIN_DISTANCE);
            filter.toDistance(MAX_DISTANCE);
            filter.distanceLabel(distanceLabel(MIN_DISTANCE, MAX_DISTANCE));
        };

        filter.showTourGlobPickOnly.subscribe(applyFilters);
        filter.fromBudget.subscribe(applyFilters);
        filter.toBudget.subscribe(applyFilters);
        filter.fromRating.subscribe(applyFilters);
        filter.toRating.subscribe(applyFilters);
        filter.toDistance.subscribe(applyFilters);
        filter.fromDistance.subscribe(applyFilters);
        filter.hotelName.subscribe(applyFilters);
        filter.stars.subscribe(applyFilters);

        $('#price-range-hotel').slider({
            range: true,
            step: 10,
            min: MIN_BUDGET,
            max: MAX_BUDGET,
            values: [MIN_BUDGET, MAX_BUDGET],
            change: function (e, ui) {
                var range = ui.values;
                filter.fromBudget(range[0]);
                filter.toBudget(range[1]);
            },
            slide: function (e, ui) {
                var range = ui.values, to = range[1];
                filter.budgetRange(range[0] + ' ' + loc.d_2timeto + ' ' + to + (to === MAX_BUDGET ? '+' : ''));
            }
        });

        $('#rating-range-hotel').slider({
            range: true,
            step: 1,
            min: MIN_RATING,
            max: MAX_RATING,
            values: [MIN_RATING, MAX_RATING],
            change: function (e, ui) {
                var range = ui.values;
                filter.fromRating(range[0]);
                filter.toRating(range[1]);
            },
            slide: function (e, ui) {
                var range = ui.values;
                filter.ratingLabel(trustYouLabel(range[0], range[1]));
            }
        });

        $('#distance-range-hotel').slider({
            range: true,
            step: 100,
            min: MIN_DISTANCE,
            max: MAX_DISTANCE,
            values: [MIN_DISTANCE, MAX_DISTANCE],
            change: function (e, ui) {
                var range = ui.values;
                filter.fromDistance(range[0]);
                filter.toDistance(range[1]);
            },
            slide: function (e, ui) {
                var range = ui.values;
                filter.distanceLabel(distanceLabel(range[0], range[1]));
            }
        });

        this.hotelName.notifySubscribersImmediately = this.hotelName.notifySubscribers;
        this.hotelName.extend({ rateLimit: { timeout: 300, method: 'notifyWhenChangesStop' } });
    }

    self.filters = new FilterBoxViewModel();
    self.mapFilters = new MapFilterBoxViewModel(map, $('#searchMap'));


    function buildHotelViewModel(hotel) {

        hotel.marker = null;
        hotel.markerZIndex = 1;

        hotel.trustYouRate = trustYouScoreToRate(hotel.trustYouScore);

        hotel.price = ko.pureComputed(function () {
            return PortalV3.convertCurrency(this.baseCurrency, PortalV3.currencyCode(), this.basePrice, 2);
        }, hotel);

        hotel.priceText = ko.pureComputed(function () {
            var price = this.price().toString(),
                result = price + ' ' + PortalV3.currencyCode();

            if (this.marker) {
                this.marker.setLabel(price);
            }

            return result;
        }, hotel);

        hotel.pricePerNightText = ko.pureComputed(function () {
            return PortalV3.convertCurrency(this.baseCurrency, PortalV3.currencyCode(), this.basePricePerNight, 2) + ' ' + PortalV3.currencyCode();
        }, hotel);

        hotel.imageUrl = hotel.imageUrl || '/Views/CustomViews/4/style/img/no-image.png';

        hotel.starsHtml = '';
        var starsLimit = Math.ceil(hotel.stars);
        for (var i = 1; i <= starsLimit; ++i) {
            hotel.starsHtml += (i > hotel.stars) ? '<span class="half">★</span>' : '<span>★</span>';
        }

        hotel.distanceToCenterText = hotel.distanceToCenter
            ? (hotel.distanceToCenter * 1.0 / 1000).toFixed(2) + ' ' + loc.d_2km + ' ' + loc.d_2fromthecenter
            : '';

        hotel.activateMarker = function () {
            var marker = hotel.marker;
            marker.setIcon(activeMarkerIcon);
            hotel.markerZIndex = marker.getZIndex();
            marker.setZIndex(google.maps.Marker.MAX_ZINDEX);
        };

        hotel.deactivateMarker = function () {
            var marker = hotel.marker;
            marker.setIcon(markerIcon);
            marker.setZIndex(hotel.markerZIndex);
        };

        hotel.openInfoWindow = function () {
            var content = '<div class="mapInfoWin"><a href="' + hotel.seoUrl + '"><img src="' + hotel.imageUrl + '"/></a><p><strong>'
                + hotel.title + '</strong><span>' + hotel.priceText() + '</p></div>';

            infoWindow.setContent(content);
            infoWindow.open(map, hotel.marker);
        };

        hotel.scrollTo = function () {
            var $hotel = $('a[href="' + hotel.seoUrl + '"]').closest('li.win');
            var top = $hotel.offset().top;

            $('body,html').animate({ scrollTop: top - 200 }, 300, function () {
                $hotel.removeClass('highlight');
            });

            $hotel.addClass('highlight');
        };

        hotel.mouseOver = function () {
            if (hotel.marker) {
                hotel.activateMarker();
                hotel.openInfoWindow();
            }
        };

        hotel.mouseOut = function () {
            if (hotel.marker) {
                infoWindow.close();
                hotel.deactivateMarker();
            }
        };

        return hotel;
    }
}

